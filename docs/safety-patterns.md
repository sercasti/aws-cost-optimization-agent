# Safety patterns

Four reusable patterns the agent applies to irreversible operations. Each one was developed in response to a specific class of mistake. Each is worth more than the dollar value of any single Quick Win it slows down.

## Pattern 1: Pre-delete safety snapshot

**When**: about to delete an EBS volume, RDS instance, or any resource where the source data is not also archived elsewhere AND deletion is irreversible.

**Why**: even when the audit shows a resource has been unused for years and the source snapshot has been cleaned up, the cost of a fresh safety snapshot is small (~$0.05/GB/mo for EBS, billed only on changed blocks) and the recoverability is total.

**How**:

```bash
aws ec2 create-snapshot \
  --volume-id $VOL \
  --description "Pre-delete safety snapshot $(date +%Y-%m-%d), reason: ..." \
  --tag-specifications "ResourceType=snapshot,Tags=[
    {Key=Purpose,Value=pre-delete-safety},
    {Key=OriginalVolumeId,Value=$VOL},
    {Key=ReviewBy,Value=$(date -d '+30 days' +%Y-%m-%d)},
    {Key=CreatedBy,Value=cost-engagement-$(date +%Y-%m)}
  ]"

# Wait for completed state, then delete original
aws ec2 wait snapshot-completed --snapshot-ids $SNAP
aws ec2 delete-volume --volume-id $VOL
```

**Cost during the safety window**: roughly 50% of the original resource cost (the snapshot stores the data but not the provisioned IOPS or compute). Net savings during the window: 50% of full savings.

**Review date**: tag the snapshot with a 30-day `ReviewBy`. At that date either delete the snapshot (full savings now realized) or extend the window if the owner needs more confidence.

## Pattern 2: Owner pre-approval is not IAM grant

**When**: an owner has agreed in writing or verbally that an action should proceed. The action is irreversible or politically visible.

**The trap**: "Alexei said go ahead with the account closures, so I'll run `aws organizations close-account` now." Then IAM blocks because `organizations:CloseAccount` is not on the user's policy.

**The fix**: at runtime, treat operational consent and IAM grant as separate prerequisites. Both must be true before acting.

**Practical check**:

```bash
# Operational: have we discussed this in the last 7 days?
# Documented in: open-questions/for-account-owners.md
# Last touch: <check timestamp>

# IAM: dry-run if the API supports it
aws ec2 delete-vpc --dry-run --vpc-id $V --region $R --profile $P
# If this returns "DryRunOperation succeeded" you have the permission.
# If it returns "UnauthorizedOperation" stop and either escalate IAM or ask the owner to run it.
```

For APIs that don't support dry-run (Organizations close-account, most account-level ops), the agent's policy is: attempt the call, accept `AccessDeniedException` as a non-failure outcome, surface to the user with both unblock options (owner runs it OR owner grants permission).

## Pattern 3: TF-PR draft with safety note (auto-apply repos)

**When**: the customer's IaC pipeline auto-applies on merge to main. You don't have backend env vars or the IaC role trust policy to run `terraform plan` locally. Standard PR workflow is "plan before merge," which is impossible here.

**The pattern**:

1. Branch from fresh `origin/main`. Apply the minimal diff per the runbook.
2. Push the branch.
3. Open the PR as a **draft**.
4. PR body includes the **expected plan output shape**: which modules, which fields, in-place modify or replacement. Reviewer compares actual plan to expected.
5. Mark "ready for review" only after a maintainer with backend access runs the plan and confirms the shape matches.
6. Squash-merge triggers the pipeline.

Per-PR description template:

```markdown
## Summary
<one paragraph, business outcome focused>

## Diff
<file count, line count, what's changed semantically>

## Safety: local plan not feasible
Backend uses partial config; pipeline injects `TF_BUCKET` / `TF_REGION` / `TF_TABLE` from CodeBuild env vars not in source. Provider assumes a role whose trust policy only allows the CodeBuild service role. Keeping draft until plan validation.

## Expected plan output (all 3 workspaces)
```
~ module.X.aws_Y.this[0]
    field: "old_value" -> "new_value"
# Plan: 0 to add, N to change, 0 to destroy
```

## Reviewer instruction
Run plan against this branch in CodeBuild (TF_ACTION=PLAN). Confirm output matches the shape above. If you see DESTROY, REPLACE, or unrelated resource changes, do NOT merge.

## Rollback
Revert PR and re-apply. Change is in-place; no data loss.
```

This shifts the safety gate from "I ran the plan" to "you ran the plan and we agreed it was clean." Slower per PR but no surprises in production.

## Pattern 4: Verification gate before destructive recommendations

**When**: about to add a finding to the savings backlog that says "delete X, save $Y/mo." X is some shared infrastructure (RDS Proxy, VPC endpoint, ElastiCache cluster).

**The trap**: classifying X as "unused" based on a single signal (low connection count, low traffic, etc.) and recommending deletion. Reality is invariably weirder.

**The 4-source verification gate** for any "this resource is unused" claim:

1. **Lambda environment variables**. `aws lambda list-functions` returns env vars directly. Grep for the resource's DNS endpoint, ARN, or identifier.
2. **ECS task definitions**. For each active task family, `describe-task-definition` returns container env vars and secrets references. Grep there.
3. **SSM Parameter Store**. `describe-parameters` then `get-parameters --with-decryption`. Grep values.
4. **Secrets Manager**. List secrets, fetch values, grep.

Only when all 4 sources return zero hits for the target's DNS / endpoint / ARN can the resource be reclassified as deletable. Even then, "deletable" means "next step is owner confirmation," not "the agent proceeds autonomously."

The agent's own audit failed this gate once during the engagement that birthed this repo. We claimed 4 RDS proxies were verifiably unused. The Lambda check was correct. The ECS check silently returned 0 hits due to a Windows filename corruption bug (CRLF in CLI output created files with stray Unicode in the names, and `grep -r` skipped them). The proxies were actually load-bearing for the production OpenFGA authorization service. The next layer of safety (pre-investigation grep across all application Terraform repos) caught the mistake before any PR was opened.

Lesson: a 4-source gate is the minimum. When any source returns "all clear," ask "what's the failure mode that would make all-clear a false positive?" before trusting it.
