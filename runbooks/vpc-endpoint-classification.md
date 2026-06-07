# Runbook: VPC interface endpoint classification

**Goal**: classify VPC interface endpoints across multiple accounts into safe-to-delete vs. must-keep buckets. The audit and the deletion are different operations and each has its own failure modes.

**Typical capture**: highly variable. Some accounts have $0 of cleanup; some have $500-1500/mo of legitimate orphan endpoints.

**Risk profile**: high. VPC endpoints can be requester-managed (AWS-owned, not CLI-deletable) and others may be load-bearing despite low BytesProcessed signal.

## Why this is its own runbook

VPC interface endpoints look simple. They have a service name, a VPC ID, security groups, subnets. The audit "find endpoints that look idle and delete them" sounds straightforward.

It is not. The engagement that built this runbook hit two failure modes back-to-back:

1. "Orphan" endpoints were actually RDS Proxy infrastructure (see `docs/failure-modes.md` entry 2).
2. The Lambda env-var check that should have caught this was disabled by a Windows path-handling bug.

This runbook encodes the workarounds for both.

## The 5-bucket classification

Every interface endpoint falls into exactly one bucket:

| Bucket | Definition | Action |
|---|---|---|
| 1. TF-managed Interface | Declared in IaC (Terraform, CloudFormation, CDK) | Never delete via CLI. Goes through IaC PR. |
| 2. CF-managed | Has `aws:cloudformation:stack-name` tag | Never delete via CLI. Either remove via stack update or escalate to stack owner. |
| 3. ClickOps ACTIVE | > 1 MB BytesProcessed in last 30 days | Keep. Active workload. |
| 4. ClickOps IDLE + PrivateDnsEnabled=true | < 1 MB in 30 days, private DNS enabled | Safe deletion candidate. Consumers use the standard service DNS, which falls back to public endpoint automatically. |
| 5. ClickOps IDLE + PrivateDnsEnabled=false | < 1 MB in 30 days, private DNS disabled | Needs extra verification. Consumers may have hardcoded `vpce-xxxx.svc...` DNS that breaks on deletion. |

Bucket 5 is where most of the danger lives.

## Phase 0a: Terraform / IaC ownership check

Grep for `aws_vpc_endpoint` in every TF repo the customer manages. Anything declared there is bucket 1. Off-limits for CLI delete.

```bash
for repo in $(ls customer-iac-repos/); do
  echo "=== $repo ==="
  find customer-iac-repos/$repo -name "*.tf" -type f \
    | xargs grep -Hn "aws_vpc_endpoint" 2>/dev/null
done
```

If any repo is not locally available, ask the customer to grep externally and paste results. Don't assume "no entries" without evidence.

## Phase 0b: CloudFormation ownership check (per env)

```bash
aws ec2 describe-vpc-endpoints --region $REGION --profile $PROFILE \
  --query 'VpcEndpoints[?Tags!=null] | [?contains(Tags[].Key, `aws:cloudformation:stack-name`)].[VpcEndpointId, ServiceName, Tags[?Key==`aws:cloudformation:stack-name`].Value | [0]]' \
  --output table
```

CF-tagged endpoints are bucket 2.

## Phase 1: Per-env inventory + traffic classification

For each env (dev, uat, prod, etc.):

```bash
PROFILE=...
REGION=...
START_TIME=$(date -u -d '30 days ago' +%Y-%m-%dT%H:%M:%SZ)
END_TIME=$(date -u +%Y-%m-%dT%H:%M:%SZ)

aws ec2 describe-vpc-endpoints --region $REGION --profile $PROFILE \
  --output json > /tmp/endpoints-$PROFILE.json

jq -r '.VpcEndpoints[] | select(.VpcEndpointType=="Interface") | .VpcEndpointId' /tmp/endpoints-$PROFILE.json \
  | while read EP_ID; do
      BYTES=$(aws cloudwatch get-metric-statistics \
        --namespace AWS/PrivateLinkEndpoints \
        --metric-name BytesProcessed \
        --dimensions "Name=VPC Endpoint Id,Value=$EP_ID" \
        --start-time $START_TIME --end-time $END_TIME \
        --period 86400 --statistics Sum \
        --region $REGION --profile $PROFILE --output json 2>/dev/null \
        | jq '[.Datapoints[].Sum] | add // 0')
      echo "$EP_ID $BYTES"
    done > /tmp/traffic-$PROFILE.txt
```

**Caveat**: AWS-native PrivateLink endpoints (ElastiCache Serverless, RDS Proxy, etc.) often don't emit BytesProcessed at all. A zero result means "no signal," not "no use." This is critical.

## Phase 2: Pre-deletion verification (bucket 5 only)

For each bucket-5 candidate, before recommending deletion, run a **mandatory SG ownership check**:

```bash
aws ec2 describe-security-groups --region $REGION --profile $PROFILE \
  --group-ids <SG_IDS_FROM_ENDPOINT> \
  --query 'SecurityGroups[].[GroupId,GroupName,Description]' --output text
```

AWS-created SGs name their owning service in `GroupName` ("`dev-records - RDS Proxy`", "`my-app - ElastiCache`"). If the SG is owned by a requester-managed AWS service, the endpoint is NOT independently deletable. Remove it from the deletion list. The endpoint will be released automatically when the upstream resource is destroyed.

This single check would have prevented the failure described in `docs/failure-modes.md` entry 2.

## Phase 3: 4-source consumer verification (bucket 4 candidates)

For bucket 4 endpoints that survived the SG ownership check, verify no consumer hardcodes the endpoint DNS:

1. **Lambda env vars** — fresh API call per function, NOT pre-saved JSON:
   ```bash
   for fn in $(aws lambda list-functions --region $REGION --profile $PROFILE --query 'Functions[].FunctionName' --output text); do
     aws lambda get-function-configuration --function-name "$fn" --region $REGION --profile $PROFILE \
       --query 'Environment.Variables' --output json | grep -F "$ENDPOINT_DNS"
   done
   ```

2. **ECS task definitions** — fresh API call per family:
   ```bash
   for fam in $(aws ecs list-task-definition-families --status ACTIVE --region $REGION --profile $PROFILE --query 'families[]' --output text | tr '\r' ' '); do
     aws ecs describe-task-definition --task-definition "$fam" --region $REGION --profile $PROFILE \
       --output json | grep -F "$ENDPOINT_DNS"
   done
   ```

3. **SSM Parameter Store** (`MSYS_NO_PATHCONV=1` on Windows):
   ```bash
   aws ssm describe-parameters --region $REGION --profile $PROFILE --query 'Parameters[].Name' --output text \
     | tr '\t\r' '\n\n' | xargs -n 10 aws ssm get-parameters --names --with-decryption --region $REGION --profile $PROFILE --output json \
     | jq -r '.Parameters[].Value' | grep -F "$ENDPOINT_DNS"
   ```

4. **Secrets Manager**:
   ```bash
   for s in $(aws secretsmanager list-secrets --region $REGION --profile $PROFILE --query 'SecretList[].Name' --output text); do
     aws secretsmanager get-secret-value --secret-id "$s" --region $REGION --profile $PROFILE --query 'SecretString' --output text \
       | grep -F "$ENDPOINT_DNS"
   done
   ```

**Pass criteria**: all 4 sources return zero hits.

## Phase 4: Deletion (with per-env confirmation gate)

Bucket 4 candidates that passed Phase 3 are deletion-eligible. Even then:

1. Announce per-env: "About to delete N endpoints in $ENV, total potential savings ~$X/mo. Confirm."
2. Wait for human confirmation.
3. Delete per-endpoint with the AWS CLI:
   ```bash
   aws ec2 delete-vpc-endpoints --vpc-endpoint-ids "$EP_ID" --region $REGION --profile $PROFILE
   ```
4. **Failure to expect**: `InvalidParameter: Operation is not allowed for requester-managed VPC endpoints`. If this fires, the upstream classification was wrong. STOP, do not proceed to other envs, re-run Phase 2 SG ownership check with the failing endpoint's SGs.
5. Post-delete smoke test: monitor NAT Gateway BytesOutToDestination for the env's NAT. A spike means traffic that was routing through the endpoint is now routing through NAT, which is expected, but the magnitude should be predictable. A 10x spike means a consumer was actually using the endpoint and your verification missed it.
6. Pause 10 minutes between envs. Customer reports of any issue in dev: stop, do not proceed to uat/prod.

## What this captured in practice

Engagement that built this runbook: initial estimate was $885/mo across "60+ Interface endpoints across 3 envs." After methodology applied:

- ~30 endpoints were bucket 1 (TF-managed), no action.
- ~5 were bucket 2 (CF-managed), no action.
- ~20 were bucket 3 (active traffic), keep.
- Initial scan claimed 45 as bucket 5 candidates for deletion. SG ownership check revealed they were RDS Proxy infrastructure (bucket 4 of "AWS requester-managed"), folded into the upstream RDS Proxy conversation, not deleted independently.

Net direct CLI deletion captured: $0. Net opportunity rolled into other tracks: ~$1000/mo (the RDS Proxy task captured the network cost as part of proxy deletion).

The story isn't "we deleted N endpoints." It's "we prevented a customer outage by classifying correctly before acting."
