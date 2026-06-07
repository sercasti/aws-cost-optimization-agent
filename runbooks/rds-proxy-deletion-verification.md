# Runbook: RDS Proxy deletion verification

**Goal**: determine whether an RDS Proxy is actually unused before recommending deletion. Avoid the "delete this idle proxy" recommendation that breaks production because a Lambda function references it via env var.

**Typical capture**: highly variable. A small set of truly idle proxies can be $50-300/mo; a wrongly-classified active proxy can be a production outage.

**Risk profile**: very high. The cost of a wrong "deletable" classification on a proxy that backs OpenFGA, the API tier, or any other shared infrastructure is a production incident.

## Why this is its own runbook

RDS Proxy is the most commonly mis-classified resource in cost optimization scans. The standard signals look right:

- Average connections per day is near zero.
- BytesProcessed on the requester-managed VPC endpoint reports 0 (because AWS doesn't emit per-endpoint metrics for this service).
- Idle hours on the proxy itself look high.

But these signals don't measure whether the proxy is **wired into** consumers, only whether it's currently **receiving** traffic. Consumers cache connections, use the proxy DNS as their primary endpoint, and may sit idle for minutes between writes. A proxy with 2 connections/day might still be the OpenFGA datastore that gates every API request.

## Phase 1: Per-proxy DNS endpoint inventory

```bash
aws rds describe-db-proxies --region $REGION --profile $PROFILE \
  --query 'DBProxies[].[DBProxyName,Endpoint,Status]' --output text \
  > /tmp/proxies-$PROFILE.txt
```

Plus the read endpoint per proxy:

```bash
aws rds describe-db-proxy-endpoints --db-proxy-name "$PROXY_NAME" --region $REGION --profile $PROFILE \
  --query 'DBProxyEndpoints[?TargetRole==`READ_ONLY`].Endpoint' --output text
```

Build a pattern file with every proxy's write and read DNS hostnames:

```
my-proxy.proxy-XXXXX.us-west-2.rds.amazonaws.com
my-proxy.endpoint.proxy-XXXXX.us-west-2.rds.amazonaws.com
other-proxy.proxy-XXXXX.us-west-2.rds.amazonaws.com
...
```

## Phase 2: 4-source consumer scan

For each proxy DNS hostname, scan four consumer surfaces. **Critical**: use direct API calls, NOT pre-saved JSON files. The Windows MSYS filename corruption bug (see `docs/failure-modes.md` entry 1) silently breaks `grep -r` on saved files.

### Lambda env vars

```bash
aws lambda list-functions --region $REGION --profile $PROFILE --output json \
  | jq -r '.Functions[] | select(.Environment.Variables != null) | .FunctionName as $f | .Environment.Variables | to_entries[] | "\($f)\t\(.key)\t\(.value)"' \
  > /tmp/envvars-$PROFILE.tsv

grep -F -f /tmp/proxy-patterns.txt /tmp/envvars-$PROFILE.tsv
```

### ECS task definitions (DO NOT use saved files on Windows)

```bash
for fam in $(aws ecs list-task-definition-families --status ACTIVE --region $REGION --profile $PROFILE \
              --query 'families[]' --output text | tr '\t\r' '\n\n'); do
  aws ecs describe-task-definition --task-definition "$fam" --region $REGION --profile $PROFILE \
    --output json | jq -r '.taskDefinition.containerDefinitions[].environment // [] | .[] | "\(.name)=\(.value)"' \
    | grep -F -f /tmp/proxy-patterns.txt && echo "  HIT in family $fam"
done
```

### SSM Parameter Store

```bash
export MSYS_NO_PATHCONV=1   # critical on Windows
aws ssm describe-parameters --region $REGION --profile $PROFILE \
  --query 'Parameters[].Name' --output text | tr '\t\r' '\n\n' | grep -v '^$' \
  | xargs -n 10 aws ssm get-parameters --names --with-decryption \
      --region $REGION --profile $PROFILE --output json \
  | jq -r '.Parameters[] | "\(.Name)=\(.Value)"' \
  | grep -F -f /tmp/proxy-patterns.txt
```

### Secrets Manager

```bash
for s in $(aws secretsmanager list-secrets --region $REGION --profile $PROFILE \
            --query 'SecretList[].Name' --output text); do
  aws secretsmanager get-secret-value --secret-id "$s" --region $REGION --profile $PROFILE \
    --query 'SecretString' --output text 2>/dev/null \
    | grep -F -f /tmp/proxy-patterns.txt && echo "  HIT in secret $s"
done
```

## Phase 3: Pre-investigation IaC grep (mandatory)

Before any proxy makes it onto a deletion list, grep every application Terraform repo for references to the proxy's outputs:

```bash
grep -r "proxy_endpoint\|rds-proxy.*outputs\|rds_proxy.*read\|rds_proxy.*write" \
  customer-iac-repos/ --include="*.tf"
```

Common patterns in app TF:

```hcl
locals {
  rds_xxx_proxy_write = data.terraform_remote_state.stateful.outputs.rds-xxx-proxy.proxy_endpoint
  rds_xxx_proxy_read  = data.terraform_remote_state.stateful.outputs.rds-xxx-proxy.db_proxy_endpoints.read_only.endpoint
}
```

If any of these surface for the proxy you're about to recommend deleting, the proxy is load-bearing. Stop.

## Phase 4: Per-proxy verdict

After Phases 1-3:

- **0 hits across all 4 sources AND 0 IaC references**: candidate for deletion. Continue to owner conversation.
- **Any hits in any source OR any IaC references**: keep. Do not include in the deletion recommendation. Note the consumers in the audit doc as load-bearing context.

The engagement that built this runbook ran the verification against 22 RDS Proxies across 3 envs. Initial Lambda-only audit reported "18 are load-bearing, 4 are clean." Pre-investigation IaC grep (Phase 3) revealed 3 of the "clean 4" were wired into ECS services as the OpenFGA authorization datastore. Lambda count was correct; ECS count was wrong because the offline grep silently failed. Net: 1 of 22 proxies was genuinely deletable, not 4. Estimated savings dropped from "$150-240/mo" to "$30-40/mo, below pursuit threshold."

The discovery happened during pre-investigation. No PR was opened. No customer impact.

## Phase 5: Owner conversation (if any candidates survive)

If after all phases a deletion candidate exists, frame the question carefully:

```
Found RDS Proxy $NAME with no detected consumers across Lambda env vars, 
ECS task defs, SSM params, Secrets Manager, or any IaC references. 

Avg connections: $N/day over 30 days.
Estimated savings if deleted: ~$X/mo.

Caveats:
- We checked all consumer surfaces we could see. We did not check direct application code 
  for hardcoded DNS strings (would require source code access).
- AWS Backup-managed AMIs of any consumer instance could have stale env vars we can't see.

Want me to proceed, hold for a week, or skip entirely?
```

The "hold for a week" option is the right default when the dollar value is below $100/mo. The cost of a wrong call here is far higher than the savings.

## What this captured in practice

Engagement that built this runbook: 1 of 22 RDS Proxies genuinely deletable, ~$30/mo. The other 21 were correctly classified as load-bearing through this exact methodology. The $2,466/mo "all proxies deletable" claim from the original audit was retracted from the customer-facing savings tracker before any PR was opened.

The story isn't "we found and deleted N proxies." It's "we caught a multi-environment outage in pre-investigation."
