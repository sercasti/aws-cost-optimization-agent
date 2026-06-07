# Runbook: Orphan snapshot sweep (RDS + EBS, cross-account)

**Goal**: identify and delete RDS manual snapshots and EBS snapshots whose source resources no longer exist, across multiple AWS accounts.

**Typical capture**: $100-500/mo for moderately sized portfolios, $1,000-5,000/mo for portfolios with heavy AMI accumulation.

**Risk profile**: medium. Snapshot deletion is irreversible. The agent uses safety classification before deleting.

## Pre-flight

- AWS CLI authenticated against each target account (separate profile per account).
- ReadOnlyAccess or equivalent for the scan phase.
- The role used for deletion needs `rds:DeleteDBSnapshot` and `ec2:DeleteSnapshot`.

## Phase 1: Scan

### RDS manual snapshots whose source DB is gone

For each account profile:

```bash
aws rds describe-db-snapshots \
  --snapshot-type manual \
  --region $REGION --profile $PROFILE \
  --query 'DBSnapshots[].[DBSnapshotIdentifier,DBInstanceIdentifier,SnapshotCreateTime,AllocatedStorage]' \
  --output text \
  | while IFS=$'\t' read -r snap dbid created storage; do
    if aws rds describe-db-instances --db-instance-identifier "$dbid" \
         --region $REGION --profile $PROFILE >/dev/null 2>&1; then
      echo "KEEP: $snap (source $dbid still exists)"
    else
      echo "ORPHAN: $snap (${storage}GB, source $dbid gone, created $created)"
    fi
  done
```

Per-snapshot cost: RDS manual snapshots are billed at $0.095/GB/mo at standard pricing.

### EBS snapshots: AMI-dereg orphans only (safest classification)

Some EBS snapshots back AMIs. Some are taken manually for backup. The cleanest "orphan" classification: snapshot whose `VolumeId` is gone AND whose snapshot is not referenced by any current AMI.

```bash
# All snapshots owned by self
aws ec2 describe-snapshots --owner-ids self \
  --region $REGION --profile $PROFILE \
  --query 'Snapshots[].[SnapshotId,VolumeId,VolumeSize,StartTime,Description]' \
  --output text > /tmp/snaps.tsv

# Snapshots referenced by any AMI
aws ec2 describe-images --owners self --region $REGION --profile $PROFILE \
  --query 'Images[].BlockDeviceMappings[].Ebs.SnapshotId' \
  --output text | tr '\t' '\n' | sort -u > /tmp/ami-snaps.txt

# Current EBS volumes
aws ec2 describe-volumes --region $REGION --profile $PROFILE \
  --query 'Volumes[].VolumeId' --output text \
  | tr '\t' '\n' | sort -u > /tmp/vols.txt

# Orphan = source vol gone AND not referenced by any AMI
while IFS=$'\t' read -r snap vol size start desc; do
  [ -z "$snap" ] && continue
  grep -qx "$vol" /tmp/vols.txt && continue
  grep -qx "$snap" /tmp/ami-snaps.txt && continue
  echo "ORPHAN-EBS: $snap (${size}GB, src $vol gone) | $desc"
done < /tmp/snaps.tsv
```

EBS snapshot cost: ~$0.05/GB/mo, BILLED ON CHANGED BLOCKS ONLY. Nominal size overstates billing significantly when snapshots derive from the same source.

## Phase 2: Classify and gate

**Auto-delete eligible**:

- RDS snapshots whose source DB has been gone for more than 1 year AND snapshot is 1+ years old.
- EBS snapshots that are AMI-dereg orphans (caught by Phase 1 classification above).

**Requires owner confirmation before delete**:

- Recent RDS snapshots (< 6 months old) even if source is gone. Could be intentional pre-decommission backup.
- EBS snapshots with names suggesting deliberate backup intent ("Backup", "Pre-deploy", "Pre-migration", etc.).
- Anything > 1 TB nominal even if "obvious orphan." Worth a 30-second confirmation.

**Never delete via this runbook**:

- Snapshots referenced by ANY AMI (handled in `unused-ami-audit.md`).
- Snapshots with the `aws:backup:source-resource` tag (managed by AWS Backup, see that service).
- Snapshots in any account where the engagement hasn't explicitly received cleanup pre-approval.

## Phase 3: Delete (auto-eligible only)

For each auto-eligible RDS snapshot:

```bash
aws rds delete-db-snapshot \
  --db-snapshot-identifier "$snap" \
  --region $REGION --profile $PROFILE
```

For each auto-eligible EBS snapshot:

```bash
aws ec2 delete-snapshot \
  --snapshot-id "$snap" \
  --region $REGION --profile $PROFILE
```

**Failure modes**:

- `InvalidSnapshot.InUse`: snapshot is shared with another AMI or another consumer (e.g., a cross-region copy). Accept as a known outcome class. The audit didn't catch this because the cross-reference is not in our local view. Skip.
- `InvalidSnapshotID.NotFound`: snapshot was deleted between scan and execution. Common in active accounts. Skip.

## Phase 4: Owner confirmation batch (requires-confirmation set)

Compose one message per owner with the held items. Template:

```
Found orphan snapshots in your account that look like cleanup candidates. 
Want to send the OK before I delete, since deletion is irreversible.

[per-snapshot]
- $SNAP_ID ($SIZE_GB GB, source $SOURCE_ID gone, created $CREATED_DATE)
  Description: $DESCRIPTION

Total: $N items, ~$X/mo
Path forward: reply OK and I batch-delete, or flag any I should keep.
```

Pattern from the engagement: owners typically reply within 24h with a global OK or with one or two specific holds.

## Verification post-delete

```bash
aws rds describe-db-snapshots --snapshot-type manual ... # count should be lower by N
aws ec2 describe-snapshots --owner-ids self ... # count should be lower by M
```

For the variance report, snapshot deletion shows up in Cost Explorer line items:

- `Amazon Relational Database Service: RDS Storage` (for RDS snapshots).
- `Amazon Elastic Block Store: Snapshot Storage Cost` (for EBS).

Expect the line item to trend down within 2-3 days post-delete.

## Cross-account orchestration

The engagement that built this runbook ran the scan across 5 accounts in a single working session:

```bash
for profile in account-1 account-2 account-3 account-4 account-5; do
  region=$(...)  # per-account default region
  scan_rds $profile $region
  scan_ebs $profile $region
done
```

Each account's profile is configured once in `~/.aws/config`. Auth via `aws login --profile <name>` for browser-based IAM user flows, or via SSO if the customer is on IAM Identity Center.
