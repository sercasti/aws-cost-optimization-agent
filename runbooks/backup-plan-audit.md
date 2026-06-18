# Runbook: AWS Backup plan audit

**Goal**: detect backup plan misconfigurations that silently double storage costs or accumulate recovery points indefinitely. Separate from the AMI accumulation cleanup (see `runbooks/unused-ami-audit.md` for the execution side).

**Typical capture**: $500-5,000+/mo for accounts with daily full-AMI backup of large instances. Tecnocontrol engagement: $2,837/mo captured from retention reduction + orphan cleanup; full AMI-to-EBS-snapshot migration still in progress at $2,500/mo additional.

**Risk profile**: medium on the detection side (read-only). High on the fix side: changing a backup plan retention policy affects ALL resources in that plan's selections simultaneously.

## Why this is its own runbook

AWS Backup is one of the most expensive services in mature accounts that aren't actively managing it. Three separate failure modes appear independently:

1. **Self-copy CopyActions**: a plan rule with a `CopyAction` whose `DestinationBackupVaultArn` resolves to the same vault as `TargetBackupVaultName`. Creates a `_CopyFrom_<region>`-suffixed duplicate of every recovery point with no DR benefit. Doubles storage cost silently.
2. **No lifecycle / retention too long**: plan rules without `DeleteAfterDays` accumulate recovery points indefinitely. With daily full-AMI backups of 1 TB+ instances, this compounds fast.
3. **Full-AMI backup policy vs. EBS snapshot policy**: AWS Backup's AMI-based policy captures every volume as a full AMI copy per backup job. The EBS snapshot policy (available since 2023) captures incremental snapshots of individual volumes, reducing storage 60-80% for similar instances.

None of these are visible in the Cost Explorer line item breakdown. They all appear as "AWS Backup: Storage" and increase monotonically.

## Phase 1: Inventory all backup plans

For each account:

```bash
aws backup list-backup-plans --region $REGION --profile $PROFILE \
  --query 'BackupPlansList[].[BackupPlanId,BackupPlanName,VersionId,CreationDate]' \
  --output table
```

Then for each plan, inspect the rules:

```bash
aws backup get-backup-plan --backup-plan-id $PLAN_ID \
  --region $REGION --profile $PROFILE \
  --query 'BackupPlan.Rules[].[RuleName,TargetBackupVaultName,ScheduleExpression,Lifecycle,CopyActions]' \
  --output json
```

Key fields to extract per rule:
- `TargetBackupVaultName` - the vault this rule backs up into
- `Lifecycle.DeleteAfterDays` - null means infinite retention
- `CopyActions[].DestinationBackupVaultArn` - compare last segment to TargetBackupVaultName

## Phase 2: Self-copy CopyAction detection

A self-copy exists when the destination vault ARN resolves to the same vault as the target:

```bash
aws backup list-backup-plans --profile $PROFILE --region $REGION \
  --query 'BackupPlansList[].BackupPlanId' --output text \
  | tr '\t\r' '\n\n' | while read plan_id; do
    plan_name=$(aws backup get-backup-plan --backup-plan-id "$plan_id" \
      --profile $PROFILE --region $REGION \
      --query 'BackupPlan.BackupPlanName' --output text)
    aws backup get-backup-plan --backup-plan-id "$plan_id" \
      --profile $PROFILE --region $REGION \
      --query 'BackupPlan.Rules[]' --output json \
      | jq -r --arg plan "$plan_name" --arg pid "$plan_id" '
          .[] | . as $rule |
          ($rule.CopyActions // [])[] |
          . as $ca |
          ($ca.DestinationBackupVaultArn | split(":") | last) as $dest_vault |
          if $dest_vault == $rule.TargetBackupVaultName then
            "SELF-COPY | plan=\($plan) (\($pid)) | rule=\($rule.RuleName) | vault=\($rule.TargetBackupVaultName)"
          else empty end'
  done
```

**Signal in the backup vault**: look for recovery points with names ending in `_CopyFrom_<region>` or `(Copy)`. Every such recovery point was created by a self-copy and has an identical twin without the suffix. Pure duplicate storage.

## Phase 3: No-retention detection

```bash
aws backup list-backup-plans --profile $PROFILE --region $REGION \
  --query 'BackupPlansList[].BackupPlanId' --output text \
  | tr '\t\r' '\n\n' | while read plan_id; do
    aws backup get-backup-plan --backup-plan-id "$plan_id" \
      --profile $PROFILE --region $REGION \
      --query 'BackupPlan.{Name:BackupPlanName,Rules:Rules[].[RuleName,Lifecycle.DeleteAfterDays]}' \
      --output json \
      | jq -r '.Name as $plan | .Rules[] |
          if .[1] == null then
            "NO-RETENTION | plan=\($plan) | rule=\(.[0])"
          else
            "OK | plan=\($plan) | rule=\(.[0]) | delete_after=\(.[1])d"
          end'
  done
```

Any `NO-RETENTION` line is a vault-grows-forever situation. Priority correlates with how large the backed-up resources are.

## Phase 4: Vault sizing

Count recovery points and estimate cost before touching anything:

```bash
VAULT_NAME="your-vault-name"
ACCOUNT_ID=$(aws sts get-caller-identity --profile $PROFILE --query Account --output text)

aws backup list-recovery-points-by-backup-vault \
  --backup-vault-name "$VAULT_NAME" \
  --profile $PROFILE --region $REGION \
  --query 'RecoveryPoints[].[RecoveryPointArn,BackupSizeInBytes,CreationDate,ResourceType]' \
  --output json \
  | jq '[.[] | {arn:.[0], gb:(.[1]/1073741824), created:.[2], type:.[3]}]
        | {total_gb:(map(.gb) | add), count:length, types:(map(.type) | group_by(.) | map({type:.[0], count:length}))}'
```

Cost estimate: total_gb × $0.05/mo (EBS snapshot storage rate, before dedup).

The actual billed amount is in Cost Explorer under `AWS Backup: Storage`. Pull that line item for the account and compare to the nominal estimate. If nominal is 3-5x billed, snapshots have high block-level dedup (expected for daily backups of the same instances). If nominal matches billed, snapshots have low dedup (likely full-AMI backups of instances that change a lot between jobs).

## Phase 5: Fix self-copy CopyActions

**Prerequisite**: owner must confirm which plans to modify. Self-copy deletion is safe (the primary recovery points remain) but the owner should verify the `CopyAction` was not intentional (e.g., an unused DR attempt to a second region using the same vault by mistake vs. an actual second-region vault ARN).

To remove CopyActions from a plan rule (preserves the rule, just removes the duplicate):

```bash
# First, get the full current plan JSON
aws backup get-backup-plan --backup-plan-id $PLAN_ID \
  --profile $PROFILE --region $REGION > /tmp/current-plan.json

# Edit: remove CopyActions arrays from the offending rule(s), set DeleteAfterDays if missing
# Then update:
aws backup update-backup-plan --backup-plan-id $PLAN_ID \
  --profile $PROFILE --region $REGION \
  --backup-plan '{"BackupPlanName":"...","Rules":[...rules with CopyActions removed...]}'
```

**Important**: `update-backup-plan` replaces ALL rules in the plan. Do NOT attempt partial updates. Extract the full current rules first, edit the specific CopyActions field, then pass the full rules block back.

Verification: after update, run `get-backup-plan` again and confirm `CopyActions` is absent or empty on the modified rules. The next backup job after the update will not create the duplicate copy. Existing duplicate recovery points can be deleted manually or left to age out under the retention policy.

## Phase 6: Fix no-retention rules

Same update pattern as Phase 5. Add `"Lifecycle": {"DeleteAfterDays": 7}` (or whichever retention the owner confirms) to each `NO-RETENTION` rule.

**Critical warning before setting retention**: if the vault currently holds recovery points from the past 6 months with no retention, AWS Backup will begin deleting the oldest ones immediately after the policy is applied. Confirm the owner understands that a 7-day retention policy on a vault with 6 months of history will result in the deletion of all recovery points except the last 7 days within hours.

If the owner needs the older recovery points for compliance, agree on the archive strategy before touching the policy.

## What this does NOT do

- Delete individual recovery points directly. AWS Backup recovery points for AMI-based backups must be deleted via `backup delete-recovery-point` not via `ec2 delete-snapshot` or `ec2 deregister-image`. Direct delete of the AMI/snapshot skips the Backup catalog entry and leaves orphaned catalog metadata.
- Resolve the underlying AMI accumulation in the vault. That's in `runbooks/unused-ami-audit.md`.

## What this captured in practice

Tecnocontrol engagement (2026-06-17):

- Two plans with self-copy CopyActions detected (RespaldoDiario plan `71c7c205`, PlanCompletoRespaldoDiario plan `ab9ae8b2`).
- Both plans had their `CopyActions` removed and retention set to 7 days.
- Recovery point count was doubled by the self-copy pattern. Estimated duplicate storage released: 50% of vault size.
- Combined: $2,837/mo captured from retention reduction + orphan cleanup. Full AMI-to-EBS-snapshot migration (additional ~$2,500/mo) queued as Walk-phase item pending owner confirmation.

Key lesson: the self-copy pattern is invisible to Cost Explorer. The `_CopyFrom_<region>` suffix on recovery point names is the only signal. Without inspecting `get-backup-plan` JSON for each plan, this can accumulate for years.
