# Runbook: Cross-account hygiene sweep

**Goal**: detect residual cost-cleanup candidates across multiple AWS accounts that weren't captured in the initial per-account scans. Validate that previous cleanup work is holding.

**Typical capture**: $20-200/mo per account scanned, depending on how clean the accounts already are.

**Risk profile**: read-only scan, low-to-medium risk on the cleanup actions.

## What this scan checks

1. **Unattached EBS volumes** (state = available, no instance ID in `Attachments`).
2. **Unassociated Elastic IPs** (~$3.60/mo each at standard pricing).
3. **Idle NAT Gateways** (zero outbound bytes in 30 days, ~$32/mo each).
4. **CloudWatch alarms in INSUFFICIENT_DATA** for 30+ days (~$0.10/mo each).
5. **CloudWatch log groups without retention** (unbounded future cost).
6. **DevOps Guru coverage status** (often left enabled with no review process, ~$10-50/mo).
7. **ECR repository storage** (orphaned container images, sometimes substantial).
8. **EFS filesystems with 0 GB stored** (phantom resources, $0 cost but hygiene).

## Per-account scan template

```bash
sweep() {
  local profile=$1 region=$2 name=$3
  echo "========================================"
  echo "ACCOUNT: $name ($profile, $region)"
  echo "========================================"

  echo "--- Unattached EBS volumes ---"
  aws ec2 describe-volumes --region $region --profile $profile \
    --filters "Name=status,Values=available" \
    --query 'Volumes[].[VolumeId,Size,VolumeType,Iops,CreateTime]' --output table

  echo "--- Unassociated Elastic IPs ---"
  aws ec2 describe-addresses --region $region --profile $profile \
    --query 'Addresses[?!AssociationId].[AllocationId,PublicIp,Domain]' --output table

  echo "--- NAT Gateways (30d outbound) ---"
  for ngw in $(aws ec2 describe-nat-gateways --region $region --profile $profile \
      --filter "Name=state,Values=available" --query 'NatGateways[].NatGatewayId' --output text); do
    bytes=$(aws cloudwatch get-metric-statistics \
      --namespace AWS/NATGateway --metric-name BytesOutToDestination \
      --dimensions Name=NatGatewayId,Value=$ngw \
      --start-time $(date -u -d '30 days ago' +%Y-%m-%dT%H:%M:%SZ) \
      --end-time   $(date -u +%Y-%m-%dT%H:%M:%SZ) \
      --period 86400 --statistics Sum \
      --region $region --profile $profile \
      --output text --query 'Datapoints[].Sum' | tr '\t\r' '\n\n' | awk '{s+=$1} END {print s+0}')
    gb=$(awk "BEGIN { printf \"%.2f\", $bytes/1073741824 }")
    echo "  $ngw: 30d outbound = ${gb} GB"
  done

  echo "--- Stale alarms (INSUFFICIENT_DATA 30+ days) ---"
  aws cloudwatch describe-alarms --state-value INSUFFICIENT_DATA --region $region --profile $profile \
    --query 'MetricAlarms[?(StateUpdatedTimestamp < `'"$(date -u -d '30 days ago' +%Y-%m-%dT%H:%M:%SZ)"'`)].[AlarmName,StateUpdatedTimestamp]' \
    --output table

  echo "--- Log groups without retention ---"
  aws logs describe-log-groups --region $region --profile $profile \
    --query 'logGroups[?retentionInDays==`null`].[logGroupName,storedBytes]' --output table
}

sweep account-1-profile us-east-1 "Account 1"
sweep account-2-profile us-west-2 "Account 2"
# ... per account
```

## Classification and action

**Unattached EBS volumes**:

- < 30 days old: probably a recent aborted attach. Verify and delete with safety snapshot (see `safety-patterns.md` Pattern 1).
- > 30 days old, < 1 year: delete with safety snapshot.
- > 1 year old: aborted backup or migration residue. Delete with safety snapshot if size > 100 GB; otherwise just delete.

**Unassociated EIPs**: release directly. Reversible by allocating a new EIP if needed.

```bash
aws ec2 release-address --allocation-id $ALLOC_ID --region $REGION --profile $PROFILE
```

**Idle NAT GWs** (< 1 GB outbound in 30 days): owner conversation required before deletion. NAT GWs often back specific routes; deleting one without checking the route table can break unrelated workloads.

**Stale alarms**: bulk delete is safe. They're not firing on anything.

```bash
aws cloudwatch delete-alarms --alarm-names "$NAME_1" "$NAME_2" "$NAME_3" \
  --region $REGION --profile $PROFILE
```

**Log groups without retention**: bulk-set 30-day retention. Non-destructive (doesn't delete logs immediately; sets a future-deletion policy).

```bash
export MSYS_NO_PATHCONV=1   # critical on Windows; see failure-modes.md
aws logs describe-log-groups --region $REGION --profile $PROFILE \
  --query 'logGroups[?retentionInDays==`null`].logGroupName' --output text \
  | tr '\t\r' '\n\n' | while read lg; do
    [ -z "$lg" ] && continue
    aws logs put-retention-policy --log-group-name "$lg" --retention-in-days 30 \
      --region $REGION --profile $PROFILE
  done
```

**DevOps Guru**: check resource collection. If billing but no one in the customer org is reviewing the insights, disable.

```bash
# Tag-based coverage
aws devops-guru update-resource-collection --resource-collection \
  '{"Tags":[{"AppBoundaryKey":"devops-guru-default","TagValues":["REMOVE"]}]}' \
  --region $REGION --profile $PROFILE

# Or CloudFormation-based: stack list goes through update-resource-collection similarly
```

**ECR**: per-repo lifecycle policy via Terraform if the account is TF-managed, or direct CLI for ClickOps accounts. Usually $5-50/mo per heavily-used repo.

**EFS empty filesystems**: read-only flag, no cost, low-priority hygiene.

## What this scan validates

The hygiene sweep is also a **validation that previous cleanup work is holding**. Run it 1 week after a major Quick Wins batch. If no regrowth, the cleanup methodology is sound. If regrowth appears (e.g., 3 new orphan EBS in an account where you cleaned 30), there's a continuous source of churn (often a misconfigured CI/CD step or an unattended script) that should be flagged separately.

The engagement that built this runbook ran the sweep 8 days after the initial Quick Wins batch. Five accounts, total findings on the second sweep: 1 orphan EBS (600 GB, traced to an aborted snapshot restore), 15 stale alarms, 23 log groups without retention. Other categories were clean. Total actionable from the second sweep: $1.50/mo (the alarms) + $60/mo (the orphan EBS). The validation that previous work was holding was worth more than the dollar finding.
