# Runbook: TF-PR draft pattern (auto-apply-on-merge)

**Goal**: ship Terraform changes safely to repos whose CI auto-applies on merge to main, when you cannot run `terraform plan` locally.

**Risk profile**: high. Auto-apply pipelines turn merge into deploy. A bad PR is a bad deploy. The draft pattern below shifts the safety gate from "I plan-checked locally" to "we plan-checked together before flipping to ready-for-review."

## Why this pattern exists

Mature enterprise Terraform setups typically:

1. Use partial backend config in `main.tf`:
   ```hcl
   terraform {
     backend "s3" {}   // bucket/region/dynamodb_table injected at runtime
   }
   ```
   The real values are CodeBuild project env vars (`TF_BUCKET`, `TF_REGION`, `TF_TABLE`), not in source.

2. Use `assume_role` per workspace to a deploy role in the target account:
   ```hcl
   provider "aws" {
     assume_role { role_arn = var.pipeline_assume_role[terraform.workspace] }
   }
   ```
   The trust policy on that role allows only the CodeBuild service role.

3. Auto-apply on push to main per a build trigger.

For an outside contributor (consultant, contractor, new engineer), all three are friction. You cannot init the backend (no env vars). You cannot assume the role (not on the trust policy). And there's no pre-merge plan stage on the PR side; CI runs the plan + apply together after merge.

The draft pattern handles this by making the plan validation a shared activity, not an individual one.

## The pattern

### 1. Branch from fresh origin/main

```bash
cd customer-iac-repo
git fetch origin
git checkout -b chore/cost-$DESCRIPTOR origin/main
```

Critical: `origin/main` not local main. Stale local branches are the most common source of accidental drift in TF PRs.

### 2. Apply the minimal diff

Only files the runbook explicitly says will change. Run:

```bash
git diff --stat
```

The output should match the runbook's expected scope. If extra files appear, something is wrong.

### 3. Push and open as draft

```bash
git add <specific files only, never -A>
git commit -m "$DESCRIPTOR"
git push -u origin chore/cost-$DESCRIPTOR

gh pr create --draft --title "$TITLE" --body "$(cat <<'EOF'
... see template below ...
EOF
)"
```

`--draft` is non-negotiable. The PR cannot be merged in this state.

### 4. PR body template

```markdown
## Summary

<One paragraph explaining the change in business terms, including expected savings.>

## Diff

`N lines across M files`:
- `path/to/file_1.tf`: <what changed>
- `path/to/file_2.tf`: <what changed>

## Safety: local plan not feasible

I do not currently have:
- Backend config (`TF_BUCKET`, `TF_REGION`, `TF_TABLE`) which are CodeBuild secrets not in source
- AWS role trust to assume `IaC` role in each linked account

This repo's CI runs `npm test` / lint on PRs, not `terraform plan`. The pipeline plan+apply only triggers on push to main.

**Keeping this PR in draft until a maintainer with CodeBuild access produces the plan for each workspace and confirms the output matches the expected shape below.**

## Expected plan output

For `development` workspace:
\`\`\`
# In-place modify only, no destroys, no replacements
~ module.example.aws_resource.this[0]
    field: "old_value" -> "new_value"
# Plan: 0 to add, 1 to change, 0 to destroy
\`\`\`

For `uat` workspace: same shape as development.

For `production` workspace:
\`\`\`
# Expected: NO CHANGES (production unaffected)
# Or: same shape as dev/uat depending on the runbook scope
\`\`\`

If the plan shows any DESTROY, REPLACE, or unrelated resource changes, do NOT merge. Revert and investigate.

## Test plan

- [ ] Maintainer with CodeBuild access produces plan for each workspace, confirms shape
- [ ] Convert PR from draft to ready-for-review
- [ ] Squash-merge to main → pipeline auto-applies dev → uat → prod
- [ ] Apply takes ~X min per env
- [ ] Verify expected resource state in AWS console post-apply
- [ ] Cost Explorer screenshot 7 days post-merge to validate $/mo drop

## Rollback

Revert this PR and re-apply. Change is in-place; no data loss.

## Related

- Runbook: `runbooks/<which one>`
- Engagement context: <one line>
```

### 5. Send a Slack DM to the maintainer

Same content as the PR body, shorter, addressed personally. Ask which workflow they prefer for ongoing cost-PRs:

**Option A**: maintainer shares the 3 backend env vars and adds your SSO principal to the IaC role trust policy. After that you run plans locally and paste output into each PR. Faster for ongoing flow.

**Option B**: maintainer runs `terraform plan` via CodeBuild for each PR branch and confirms the shape. Zero infra changes on their side. Slower per PR but no setup.

Either is fine. Both are reasonable. The point is to ask once and commit to a pattern for the engagement, not negotiate per-PR.

### 6. Wait, validate, flip to ready

When maintainer confirms shape:

```bash
gh pr ready $PR_NUMBER
```

Then standard squash-merge through their review process.

## What this pattern prevents

- Accidental drift in the PR diff (caught at step 2).
- Stale branches creating phantom conflicts (caught at step 1).
- Auto-apply without plan validation (caught by `--draft` at step 3).
- "We forgot to discuss the workflow and now every PR has Leonardo answering the same question" (caught at step 5).

## What this pattern does NOT prevent

- A maintainer running plan against a stale branch and missing recent main updates. Re-base if main has moved since the PR was opened.
- A wrong runbook in the first place. Pre-flight investigation (grep, ownership check, etc.) is the prior runbook's responsibility.

## What this captured in practice

Engagement that built this runbook opened 2 cost-savings TF PRs on different repos using this exact pattern. Both were in draft for 48-72 hours pending maintainer plan validation, both merged cleanly, both produced the expected savings. Total: ~$1,164/mo recurring across the two PRs.

The "Option A vs Option B" framing in step 5 was answered by the customer's tech lead within an hour: Option B (he runs the plans, no infra changes on his side). This stayed efficient because we asked once.
