# Methodology

The agent applies a 4-phase engagement model. Each phase has a different risk profile, a different verification standard, and a different output format. Conflating them is the most common way these engagements stall.

## Phase 1: Crawl (read-only, 5-7 days)

**Goal**: cost baseline + per-account inventory + open-question intake. Zero state changes.

**Outputs**:

- Cost Explorer snapshot at engagement start (the baseline against which variance is measured later).
- Per-account inventory in a consistent shape: compute, storage, networking, data services, observability.
- Open-question list per owner. Every "is this still needed?" gets a name attached.

**Failure modes**:

- Skipping the cost baseline. Without a pre-state, the +7-day variance report cannot attribute savings to your work.
- Inventorying without owner mapping. Findings without an owner are findings that won't move.

**Specific scans the agent runs**:

- Stopped EC2 instances older than 30 days (EBS still bills).
- RDS instances with avg connections < 1 over 14 days.
- DMS replication instances with zero CDC datapoints over 14 days.
- Storage classes with mis-sized provisioned IOPS (io2 throttling, gp2 baseline ceilings).
- Manual RDS snapshots whose source DB no longer exists.
- AMIs whose backing snapshots no longer exist or vice versa.
- Unassociated Elastic IPs (~$3.60/mo each at standard pricing).
- NAT Gateways with < 1 GB outbound bytes in 30 days.
- Log groups without retention.
- DevOps Guru coverage status.

## Phase 2: Walk (Quick Wins, days 5-15)

**Goal**: capture AWS-state savings via direct CLI. State-only items, not Terraform resources.

The distinction matters. AWS-state items include manual snapshots, EIPs, DevOps Guru configuration, NAT Gateway tagging, log retention. These are managed by humans through the console or CLI, never by IaC. Direct CLI changes here create no drift.

Terraform-managed items (RDS instance sizes, ECS service counts, security group rules) are not Quick Wins material under this methodology. They go through Phase 3.

**Quick Wins criteria** (must satisfy all):

1. Low effort. Single command or small batch.
2. Low risk. Failure mode is recoverable in minutes, not hours.
3. Reversible. AWS provides an undo path.
4. Owner-approved. Operational consent AND IAM permission both confirmed.

Platform migrations and architecture changes do not qualify as Quick Wins regardless of dollar value.

**Failure modes**:

- Treating "pre-approved" as "go anytime without checking." Pre-approval ages. Re-confirm before acting if the conversation was more than 7 days ago.
- Treating "IAM granted" as "operational consent." A user with `organizations:CloseAccount` permission can still cause political damage by closing the wrong account without warning.
- Direct CLI changes on a Terraform-managed resource. Creates state drift and the next `terraform apply` either recreates the resource or fails.

## Phase 3: Run (Terraform PRs, weeks 2-5)

**Goal**: capture savings via Terraform changes that auto-apply on merge or via PR-driven CI.

This is where most engagements break down. Common breakage:

1. Working from a stale local branch and including unrelated drift in the PR.
2. Missing the safety gate when the pipeline auto-applies on merge.
3. Lacking the ability to run `terraform plan` locally because backend config and IaC role trust are not portable.

**The agent's TF-PR pattern** addresses all three. See `runbooks/tf-pr-draft-pattern.md`.

Per-PR safety checklist:

- Branched from fresh `origin/main`, not from yesterday's local.
- Diff scope check (`git diff main..HEAD --stat`) shows ONLY files the runbook said would change.
- PR description includes the expected plan output shape ("in-place modify only, no destroys, no replacements").
- Draft status until the plan is validated by someone with infra access.

## Phase 4: Strategy (week 4+)

**Goal**: capture savings via account-level commitments. Savings Plans and Reserved Instances analyzed across the org, not per-account.

This phase requires CTO-level alignment because the commitments span 1-3 years. The agent prepares the analysis (utilization patterns, break-even calculations, commitment ladder) but the decision conversation happens with executives, not with engineers.

## What the variance report shows

7 days after the bulk of Phase 2 + early Phase 3 changes land, the agent re-pulls Cost Explorer and diffs against the Phase 1 baseline. The report attributes savings line-by-line and flags any line items that increased unexpectedly (which is almost always a sign that something downstream is now routing through a more expensive path, e.g. removing a VPC interface endpoint routes traffic through NAT, which costs more).

This is the artifact that turns "we did stuff" into "we saved you $X recurring."
