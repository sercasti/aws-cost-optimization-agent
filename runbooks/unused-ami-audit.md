# Runbook: Unused AMI audit and bulk cleanup

**Goal**: identify AMIs in customer accounts that are no longer referenced by any compute resource, deregister them, and delete their backing snapshots.

**Typical capture**: highly variable. The engagement that built this runbook saw $1,400-3,200/mo real billed savings (after dedup) from a single afternoon of execution on two accounts.

**Risk profile**: medium. Deregistration is irreversible if backing snapshots are also deleted. Owner approval is required before bulk action.

## Why AMI accumulation is the biggest single opportunity in mature accounts

Two patterns drive accumulation:

1. **Manual pre-maintenance backups**. Operators create an AMI of an instance before a risky change, name it with a timestamp ("ServerName - 18 Nov 2024"), and never delete it. Over years, these accumulate to dozens or hundreds of unused AMIs per account.
2. **AWS Backup recurring rules with no retention or excessive retention**. AWS Backup will happily take a daily AMI of an instance and keep them indefinitely if the backup plan rule doesn't specify a `DeleteAfterDays`.

Pattern 1 contributes most of the storage in our experience. Pattern 2 contributes less per instance but compounds if the daily snapshot is large (1 TB+ instance volumes).

## Phase 1: Scan

For each AWS account:

```bash
# All AMIs owned by self
aws ec2 describe-images --owners self --region $REGION --profile $PROFILE \
  --query 'Images[].{Id:ImageId,Created:CreationDate,Name:Name,Size:sum(BlockDeviceMappings[?Ebs!=null].Ebs.VolumeSize)}' \
  --output json > /tmp/images.json

# AMI IDs referenced by current instances (running or stopped)
aws ec2 describe-instances --region $REGION --profile $PROFILE \
  --query 'Reservations[].Instances[].ImageId' --output text \
  | tr '\t\r' '\n\n' | sort -u > /tmp/inst-amis.txt

# AMI IDs referenced by ANY Launch Template version (not just the default)
aws ec2 describe-launch-template-versions --region $REGION --profile $PROFILE \
  --query 'LaunchTemplateVersions[].LaunchTemplateData.ImageId' --output text \
  | tr '\t\r' '\n\n' | sort -u >> /tmp/inst-amis.txt

# Combined in-use set
sort -u /tmp/inst-amis.txt | grep -v '^$' > /tmp/used.txt
```

Then compute the unused set in your language of choice:

```python
import json
images = json.load(open('/tmp/images.json'))
used = set(line.strip() for line in open('/tmp/used.txt'))
unused = [i for i in images if i['Id'] not in used]
total_gb = sum((i.get('Size') or 0) for i in unused)
print(f"{len(unused)} unused AMIs, {total_gb} GB nominal backing storage")
print(f"Estimated nominal cost: ~${total_gb * 0.05:.0f}/mo at $0.05/GB/mo list price")
print(f"Real billed cost likely 30-40% of nominal due to AWS snapshot block dedup")
```

## Phase 2: Classification

Three buckets:

| Bucket | Criterion | Action |
|---|---|---|
| Manual pre-maintenance | Description starts with "Created by CreateImage", name has timestamp pattern, created > 12 months ago | Owner confirmation, then bulk delete |
| AWS Backup managed | Name matches `AwsBackup_*` or has `aws:backup:source-resource` tag | Address via backup plan retention, NOT via direct delete |
| Cross-region copies | Description contains "Copied for DestinationAmi" | Verify destination AMI exists and is in use; if yes, delete source-region copy |

The engagement's experience was that 80%+ of accumulated storage falls into the first bucket. The AWS Backup bucket is smaller per account but worth fixing at the backup-plan level.

## Phase 3: Owner conversation

Bulk deregistration without owner approval is unacceptable on a customer account. The conversation template:

```
Found $N unused AMIs accumulating $X TB of backing storage on $ACCOUNT.

Top patterns:
- Manual backup snapshots with date stamps (ServerName - 18 Nov 2024, etc.)
- AwsBackup_* recurring AMIs of $INSTANCE_ID
- $OTHER patterns

Nominal storage cost: ~$Y/mo at list price.
Real billed cost: likely $Y/2 to $Y/3 after AWS snapshot block dedup.

Two questions:
1. Can I bulk-clean the manual backups older than 12 months?
2. For the AwsBackup recurring ones, what retention do you want? Currently the plan has no retention or excessive retention.
```

Typical owner response in practice: "delete everything older than 12 months" or "delete everything older than X (some specific date)." 

## Phase 4: Bulk dereg + snapshot delete

Per-AMI workflow:

```python
import json, subprocess

def run(cmd):
    return subprocess.run(cmd, capture_output=True, text=True)

for ami in eligible_amis:
    ami_id = ami['Id']
    # Get backing snapshots
    r = run(['aws','ec2','describe-images','--image-ids', ami_id, '--region', REGION, '--profile', PROFILE,
            '--query', 'Images[0].BlockDeviceMappings[?Ebs!=null].Ebs.SnapshotId', '--output', 'json'])
    snaps = json.loads(r.stdout) if r.returncode == 0 else []
    
    # Deregister AMI
    r = run(['aws','ec2','deregister-image','--image-id', ami_id, '--region', REGION, '--profile', PROFILE])
    if r.returncode != 0: continue
    
    # Delete each backing snapshot
    for s in snaps:
        run(['aws','ec2','delete-snapshot','--snapshot-id', s, '--region', REGION, '--profile', PROFILE])
```

**Order matters**: deregister the AMI first, then delete snapshots. Trying to delete the snapshot first will fail with `InvalidSnapshot.InUse` because the AMI still references it.

**Snapshot-sharing gotcha**: a snapshot can be referenced by multiple AMIs (often happens with cross-region copies that share the source snapshot). Snapshot delete will fail with `InvalidSnapshot.InUse` in that case. Accept as a non-failure outcome. The snapshot will become deletable after the last referencing AMI is deregistered.

## Phase 5: AWS Backup plan retention (separate workstream)

For AwsBackup_*-pattern AMIs, the root cause is the backup plan rule. The fix is at the plan level:

```bash
# Find backup plans
aws backup list-backup-plans --region $REGION --profile $PROFILE \
  --query 'BackupPlansList[].[BackupPlanId,BackupPlanName]' --output table

# Inspect the rule
aws backup get-backup-plan --backup-plan-id $PLAN_ID --region $REGION --profile $PROFILE \
  --query 'BackupPlan.Rules[].[RuleName,ScheduleExpression,Lifecycle]' --output json

# Update the rule with proper retention (sketch)
aws backup update-backup-plan --backup-plan-id $PLAN_ID --region $REGION --profile $PROFILE \
  --backup-plan '<JSON with Lifecycle.DeleteAfterDays = 30>'
```

**Important**: changing the plan rule retention affects ALL resources in the plan's selections, not just the one that caught your attention. If the plan covers 25 servers, changing from 7-day to 30-day retention multiplies storage by ~4.3x across all 25.

If you only want to change retention for one specific instance, create a new selection + new rule and move that instance to it, then leave the original alone.

## Verification and variance

Post-execution, validate via Cost Explorer line item `Amazon Elastic Block Store: Snapshot Storage Cost` for the affected accounts. Expect a measurable drop within 2-3 days. The actual drop is 30-50% of nominal (snapshot dedup), so size expectations accordingly when reporting to the CTO.

## What this captured in practice

From the engagement that built this runbook:

- Primary SaaS-application account: 81 AMIs deregistered, 168 backing snapshots deleted, ~67 TB nominal storage freed in one batch. Account owner approved "delete anything older than 12 months."
- Same account, second batch: 23 more AMIs (the "everything before Jan 1" follow-up), 54 snapshots, ~22 TB nominal.
- IoT data account: 11 AMIs deregistered, 19 snapshots, ~8 TB nominal.

Total nominal storage freed in a single afternoon: 89 TB.

Real billed savings (per Cost Explorer +7 day check): ~$1,400-3,200/mo recurring across the two accounts.
