# Failure modes catalog

The agent has a list of specific things to check for, each developed in response to an actual mistake or near-miss during the engagement that birthed this repo. Each entry has the symptom, the cause, the fix, and the lesson.

## 1. The RDS Proxy that looked deletable but was load-bearing

**Symptom**: audit reported four RDS Proxies as "verified clean across all four sources" (Lambda env vars, ECS task defs, SSM, Secrets Manager). Recommended deletion to capture ~$150-240/mo. Pre-investigation grep across application Terraform repos revealed the proxies were wired into the production authorization service as `OPENFGA_DATASTORE_URI` / `OPENFGA_DATASTORE_SECONDARY_URI`. Deleting them would have broken authz in all three environments.

**Root cause**: the ECS task definition check used `grep -r` against per-task-def JSON files saved with `aws ecs describe-task-definition`. The file names came from `jq -r` output of `aws ecs list-task-definition-families` in a bash for-loop on Windows MSYS. CRLF in the JSON output put a trailing CR on the family name string. The CR survived into the filename when files were saved, encoded as U+F00D (Private Use Area). `grep -r` on Windows silently failed to read files with those corrupt names. Zero hits returned. The verification reported "clean" when the actual data showed three of four proxies were in active use.

**Fix**:

1. When iterating shell variables sourced from JSON or AWS CLI text output, always `tr -d '\r'`.
2. Prefer direct AWS API calls (`aws ecs describe-task-definition --query 'taskDefinition.containerDefinitions[].environment'`) over disk-saved JSON. The API output goes straight to grep without filename intermediation.
3. After any "all clear" result on a destructive recommendation, ask "what failure mode would produce a false negative here?" and check at least one alternate path.

**Lesson**: a 4-source verification gate is necessary but not sufficient. The grep mechanism itself can fail silently on Windows. Live API calls or fresh re-runs in a different shell are mandatory before claiming "deletable."

## 2. The VPC endpoints that were RDS Proxy infrastructure

**Symptom**: a VPC interface endpoint audit classified 45 endpoints as "orphan" based on security-group cross-reference against active ElastiCache Serverless caches. Estimated $992/mo savings. Phase 2 deletion attempt returned `InvalidParameter: Operation is not allowed for requester-managed VPC endpoints` on all 16 attempted in the dev environment.

**Root cause**: the audit's SG cross-reference filtered only against ElastiCache Serverless caches. RDS Proxies emit the same `vpce-svc-*` PrivateLink pattern and were never checked. The "orphan" SGs were actually RDS Proxy security groups (named `*-records - RDS Proxy`, etc.). The VPC endpoints were not orphan; they were the requester-managed PrivateLink endpoints AWS auto-creates when a VPC consumes an RDS Proxy. They cannot be CLI-deleted. They're released only when the upstream RDS Proxy is deleted.

**Fix**:

1. When an SG is labeled "orphan," `aws ec2 describe-security-groups --group-ids <sg>` is a 2-second sanity check. AWS-created SGs name their owning service in `GroupName` and `Description` ("`dev-next-records - RDS Proxy`"). Read it.
2. The `vpce-svc-*` ID space is shared across AWS-managed PrivateLink services (RDS Proxy, ElastiCache Serverless, NLB-backed services). Don't assume one origin.
3. A failed deletion is not "the audit was wrong about the resource" but "the audit was wrong about the relationship." Investigate the upstream resource, not the endpoint.

**Lesson**: the $992/mo savings claim was not separate from the RDS Proxy deletion task. The endpoints are the network surface of the proxies. Capturing both as independent line items in the savings tracker creates phantom dollars that have to be removed when reality intrudes.

## 3. "Pre-approved" did not include IAM permission

**Symptom**: customer's tech lead pre-approved closing two suspended AWS accounts under their org. Operational consent confirmed in Slack. The `aws organizations close-account` calls both returned `AccessDeniedException: You don't have permissions to access this resource`.

**Root cause**: pre-approval was operational ("yes, close them") but the customer's user policy did not include `organizations:CloseAccount`. The pre-approval and the IAM grant are independent prerequisites.

**Fix**: treat them as separate gates. Operational consent unblocks the conversation. IAM grant unblocks the API. Both must be true before acting. When IAM fails, surface to the owner with both unblock options:

1. The owner runs the command themselves (they have the permission and the operational consent).
2. The owner adds the missing IAM permission to your user and you retry.

The first is usually faster.

**Lesson**: dry-run when the API supports it. For APIs that don't dry-run (most account-level operations), the agent's policy is "attempt, accept AccessDeniedException as a known outcome class, surface unblock options."

## 4. Snapshot dedup makes nominal savings overstated

**Symptom**: AMI audit found 200 TB of "unused backing snapshot storage" on a single account. At list-price $0.05/GB/mo that's ~$10,000/mo. Real billed amount turned out to be roughly 30-40% of nominal once you check the actual Cost Explorer line item.

**Root cause**: AWS only bills for unique blocks in EBS snapshot storage. Daily AMIs of the same instance share the vast majority of blocks. Nominal "size" (sum of snapshot `VolumeSize`) overstates billing dramatically when snapshots are derived from the same source over time.

**Fix**:

1. When sizing an AMI cleanup opportunity, always report nominal and "real billed estimate" as a range.
2. Pull the actual `Amazon Elastic Block Store: Snapshot Storage Cost` line item from Cost Explorer before quoting a savings number externally.
3. After the cleanup, the +7-day variance report against Cost Explorer is what attributes the actual delta. Nominal is a planning number, not a commitment.

**Lesson**: never quote nominal savings to a CTO. Always quote a range or the post-execution variance.

## 5. Low connection count is not a reliable "low criticality" signal

**Symptom**: an RDS Multi-AZ instance averaged 2 connections per day over 14 days. The audit recommended disabling Multi-AZ to save ~$140/mo. The owner replied: "this is a production database with end-client data, Multi-AZ was added in response to a past client incident, the client billing was increased to cover it."

**Root cause**: connection count and "criticality" are different dimensions. A low-frequency RDS can still hold business-critical data. And the multi-AZ premium might be funded explicitly via a customer charge increase, in which case removing it is a contract violation, not an efficiency gain.

**Fix**: add to the audit's per-RDS question list:

1. Is this passed-through to a customer in their billing?
2. Has there been a past availability incident that justified the current configuration?

Both are questions for the owner, not signals derivable from CloudWatch.

**Lesson**: cost-vs-criticality is not measurable from metrics alone. Pass-through cost arrangements are common in B2B SaaS and shift the calculus completely.

## 6. Windows Git Bash mangles paths in CLI args

**Symptom**: AWS CLI calls that pass forward-slash strings (SSM parameter names, S3 prefixes, log group names) fail with regex validation errors:

```
InvalidParameterException: Value 'C:/Users/X/AppData/Local/Programs/Git/aws/lambda/foo' at 'logGroupName' failed to satisfy constraint ...
```

**Root cause**: Git Bash on Windows (MSYS environment) auto-converts forward-slash strings that look like Unix paths into Windows paths before passing them to the executable. `/aws/lambda/foo` becomes `C:/Users/.../Git/aws/lambda/foo`.

**Fix**:

```bash
export MSYS_NO_PATHCONV=1
# now AWS CLI gets the raw string
```

Apply to any script that touches SSM, CloudWatch Logs, S3 bucket key prefixes, or other AWS resource paths that start with `/`.

**Lesson**: when AWS CLI errors look weirdly path-like on Windows, suspect MSYS first.

## 7. AwsBackup creates AMIs without expiration unless the plan specifies one

**Symptom**: scan found 7 daily 1.4 TB AMIs of the same instance, all from the past week, with names like `AwsBackup_i-XXXXXXX_*`. Initial assumption: AWS Backup retention policy is misconfigured and the AMIs are accumulating without expiry.

**Root cause** (after checking): the backup plan rule had a 7-day retention. Only 7 recovery points existed because AWS Backup correctly expires the older ones. The accumulation we were really worried about was the **manual** AMIs (created by humans before maintenance windows and never deleted), not the AWS-Backup-managed ones.

**Fix**: when inspecting AMI accumulation, classify by source first:

1. AMIs created by `CreateImage` with description "Created by CreateImage(i-...) for ami-..." are manual or scripted captures.
2. AMIs with names matching `AwsBackup_*` or tags `aws:backup:source-resource` are managed by AWS Backup. Address via the backup plan, not via direct delete (which would orphan the recovery point catalog).
3. AMIs with description "Copied for DestinationAmi" are cross-region copies. Often safe to clean once the destination is verified.

**Lesson**: not every accumulating AMI is misconfigured backups. Sometimes the configuration is correct; the accumulation is in a different bucket.

## 8. Bash variable expansion in commit messages eats dollar signs

**Symptom**: ran `git commit -m "Snapshot + delete Tecnocontrol orphan EBS, $60/mo (snapshot held 30d)"`. Resulting commit message read `Snapshot + delete Tecnocontrol orphan EBS, 0/mo (snapshot held 30d)` because bash treated `$60` as variable `$6` followed by `0`.

**Fix**: single-quote the message when it contains dollar signs, OR escape (`\$60`), OR use a heredoc.

**Lesson**: small cosmetic issue but worth catching before the commit lands.
