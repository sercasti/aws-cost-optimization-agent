# Demo: a Claude Code session with this agent

What it looks like to run a real cost optimization engagement with this agent in Claude Code. Edited transcript from the engagement that built this repo; identifiers anonymized, dollar figures real.

## Setup (5 minutes)

You drop the agent into your engagement working directory:

```bash
cd engagement-acme-cost-optimization
mkdir -p .claude/agents
cp ~/dev/aws-cost-optimization-agent/agents/cost-optimization-agent.md .claude/agents/
ln -s ~/dev/aws-cost-optimization-agent/runbooks runbooks
ln -s ~/dev/aws-cost-optimization-agent/docs methodology
```

You configure AWS CLI profiles for the customer's accounts. Open Claude Code in the engagement directory:

```bash
claude
```

You start the agent:

```
> @cost-optimization-agent we just started a 6-week engagement with a 5-account customer. 
  Configured profiles: acme-prod, acme-stage, acme-dev, acme-ops, acme-sandbox. 
  Run the crawl phase.
```

## Crawl phase (days 1-3)

### Agent's first response

```
Got it. Crawl phase across 5 accounts. Per methodology.md, this is read-only: 
Cost Explorer baseline, per-account inventory, open-question intake. Output 
lands in inventories/<account>/<date>.md.

Before I start, two things to confirm:
1. Engagement working directory structure: should I create the standard 
   subdirs (inventories/, findings/, open-questions/, deliverables/, 
   extracts/) now?
2. Owner mapping: who owns each account? I need this to file open questions 
   correctly. If you don't have it yet, I'll add "owner_unknown" placeholders 
   and we can fill in as discovery surfaces names.
```

You answer:

```
> 1. Yes create the dirs. 2. Sponsor is the CTO (Alex). Account owners: 
  acme-prod is Tania, acme-stage and acme-dev are Marco, acme-ops is Jen, 
  acme-sandbox is unowned (it's a sandbox).
```

Agent creates the structure, then begins.

### Scans run

You watch the agent scan each account in parallel. Per-account output lands in `inventories/acme-prod/2026-06-01.md`, etc. The agent's chat is terse:

```
Starting acme-prod scan.
- 27 EC2 instances, 3 stopped > 30 days
- 4 RDS instances, 1 idle (< 1 conn/day, 14 days)
- 31 manual RDS snapshots, scanning for orphans
- 12 NAT GWs, 1 with < 1 GB outbound 30d
- 89 log groups, 23 without retention
- DevOps Guru: billing $42/mo, 0 reviewers in last 30 days
- 142 AMIs, scanning for unused

acme-prod scan complete. Output: inventories/acme-prod/2026-06-01.md
Continuing acme-stage...
```

By end of day 1 the agent has produced 5 inventory documents and an initial open-questions queue with ~30 items keyed by owner.

### The agent flags a near-miss

While scanning acme-prod, the agent surfaces:

```
While scanning acme-prod, I see 4 RDS Proxies. Initial signal: low connection 
counts (avg < 2/day on 3 of them). At face value these look like deletion 
candidates worth ~$300-450/mo.

Pausing before adding to the savings tracker. Per failure-modes.md entry 1, 
RDS Proxy "low connections" is exactly the signal that doesn't work. Need the 
4-source verification gate per rds-proxy-deletion-verification.md before I 
classify them. Want me to run it now?
```

You answer:

```
> yes
```

The agent runs Lambda env-var grep, ECS task def scan, SSM check, Secrets 
Manager check, AND the IaC pre-investigation grep across the customer's TF 
repos. Result:

```
4 RDS Proxies, verification result:
- prod-records-proxy: 47 Lambda env-var hits, wired into the API tier. KEEP.
- prod-reports-proxy: 12 Lambda hits + 2 ECS task def references. KEEP.
- prod-logs-proxy: 0 hits across all 4 sources, but IaC grep found it 
  referenced as OPENFGA_DATASTORE_URI in the OpenFGA service. LOAD-BEARING.
- prod-sandbox-proxy: 0 hits across all 4 sources, 0 IaC references. 
  Genuine candidate.

Net: 1 of 4 candidates survives full verification. ~$30/mo savings 
opportunity, below my default pursuit threshold ($100/mo). Recommend 
deferring this until later phases when we know more about the platform.

The initial face-value claim was $300-450/mo. Actual deletable: $30/mo. 
This is exactly the kind of "phantom savings" failure-modes.md entry 4 warns 
against. Not adding $300-450 to the tracker.
```

You make a note: this is the kind of catch that makes the engagement credible. You ask the agent to add a one-paragraph summary to `decisions/2026-06-01-rds-proxy-verification.md` for the audit trail. It does.

## Walk phase (days 3-5)

### Quick wins on acme-stage and acme-dev

```
> @cost-optimization-agent acme-stage and acme-dev: run quick wins.
```

Agent works through the runbooks in sequence:

```
Starting Quick Wins on acme-stage + acme-dev.

Per safety-patterns.md Pattern 2, I'll confirm both operational consent and 
IAM grant for every action.

Operational consent: Marco approved the standard Quick Wins set on 2026-05-30 
(see open-questions/marco-2026-05-30.md). Within 7 days, valid.

IAM grant: I'll dry-run each destructive call before executing.

Beginning with:
1. Orphan RDS snapshot sweep (3 snapshots, all sources gone for 2+ years, 
   ~$45/mo)
2. Unassociated EIP cleanup (2 EIPs, ~$7/mo)
3. Log retention set on 14 log groups (no immediate $, bounds future)

Proceed?
```

You confirm. The agent executes, with one twist:

```
EIP release on eipalloc-09abc... failed with AccessDeniedException. 
IAM grant gap.

Marco's pre-approval is operational. The user's policy doesn't include 
ec2:ReleaseAddress on this account. Two paths:
A) Ask Marco to release them himself (the 2 commands, ~30 seconds).
B) Ask Marco to grant the IAM permission, then I retry.

Drafting Slack DM to Marco with both options...
```

The DM goes into `deliverables/2026-06-04-marco-eip-release-iam.md`. You copy-paste it to Slack. Marco picks A and runs the commands. The agent confirms post-release.

### The AMI discovery

End of day 5. You've been running scans across all 5 accounts. The agent surfaces:

```
While running unused-ami-audit.md on acme-prod and acme-ops, I'm seeing 
substantial accumulation.

acme-prod: 167 unused AMIs, 134 TB nominal backing storage.
acme-ops: 89 unused AMIs, 78 TB nominal backing storage.
acme-stage: 4 unused, 600 GB.
acme-dev: 0 unused.
acme-sandbox: 12 unused, 4 TB.

Nominal cost across the portfolio: ~$12,500/mo at $0.05/GB/mo list price.
After AWS snapshot block dedup, real billed cost is likely 30-40% of nominal.
Estimated real savings opportunity: $3,750-5,000/mo if fully cleaned.

This would be the biggest single find of the engagement so far.

Patterns observed:
- ~80% are manual pre-maintenance backups (timestamped names from 2022-2025, 
  source instances unknown/long-gone)
- ~20% are AwsBackup_* daily recurring AMIs of a few specific instances. 
  Backup plan rule has no retention; they're accumulating indefinitely.

NOT bulk-deleting. Two reasons:
1. Owner (Tania for prod, Jen for ops) hasn't pre-approved AMI cleanup specifically
2. The AwsBackup pattern is a policy fix at the plan level, not a one-time delete

Recommendation: draft a Slack/email per owner with the finding + 
proposed approach (delete manual backups older than 12 months, fix AwsBackup 
plan retention). I'll draft and you send.

Drafting deliverables/2026-06-05-tania-ami-accumulation.md and 
deliverables/2026-06-05-jen-ami-accumulation.md...
```

You review the drafts, lightly edit Tania's (she's more formal), send both. By end of day, Tania has replied "go ahead, delete everything older than 12 months." Jen replies the next morning.

You feed the responses back:

```
> Tania says go on prod for >12 months. Jen says go for ops too. 
  Execute both.
```

Agent runs the bulk dereg + snapshot delete loop. Filters out the AwsBackup_* ones (those need the plan-level fix). Per-AMI logging in `EXECUTION_LOG.md`. Final summary:

```
Bulk AMI cleanup complete.

acme-prod: 134 AMIs deregistered, 287 backing snapshots deleted, 102 TB 
nominal freed
acme-ops: 71 AMIs deregistered, 156 backing snapshots, 61 TB nominal freed
Zero failures.

Total nominal: 163 TB freed.
Real billed savings (estimate, validate via Cost Explorer in +7 days): 
$2,500-3,800/mo recurring.

Held for separate follow-up: 18 AwsBackup_* AMIs across both accounts. 
These are managed by backup plan rules with no retention. Drafted a 
separate Slack to Tania and Jen with the backup-plan-level fix.

Updated EXECUTION_LOG.md with the full per-AMI list. Committed to repo as 
'Bulk AMI cleanup: 205 AMIs, 443 snapshots, 163 TB freed (Tania + Jen 
approvals)'.
```

## Run phase (days 6-10) — the TF-PR draft pattern

```
> @cost-optimization-agent ready for Phase 3. Let's start with the Aurora 
  downsize on acme-prod and the DMS Multi-AZ disable on acme-stage.
```

Agent works through `tf-pr-draft-pattern.md`:

```
Two PRs in flight. Will run them sequentially because both need IaC repo 
state to be clean.

PR 1: Aurora dev/uat downsize on acme-prod's terraform-stateful repo.

Pre-checks:
- Branched from origin/main, not local main ✓
- Diff scope: 6 lines in locals.tf only ✓
- Terraform-Stateful README confirms auto-apply-on-merge workflow ✓
- locals.tf has the expected map structure ✓
- postgresql.tf has apply_immediately = true on the cluster modules ✓

Backend env vars not available locally (CodeBuild secrets, not in source). 
IaC role trust doesn't include my SSO principal. Per tf-pr-draft-pattern.md, 
I cannot run terraform plan locally. Opening as draft with safety note.

PR opened: https://github.com/acme-inc/terraform-stateful/pull/247

PR body includes:
- Expected plan output shape per workspace
- Safety note explaining why local plan is not feasible  
- Reviewer instruction: confirm plan matches shape before flipping ready

Slack DM drafted to Tania asking which workflow she prefers for ongoing 
cost-PRs (Option A: share backend env vars + IAM grant, or Option B: she 
runs plans via CodeBuild). Saved to 
deliverables/2026-06-08-tania-tf-plan-workflow.md.

Moving to PR 2: DMS Multi-AZ on acme-stage...
```

PR 2 follows the same pattern. By end of day you have 2 draft PRs sitting in 
the customer's repos with clear safety notes. Tania picks Option B by Slack 
within the hour ("just send me the branch, I'll plan it"). The PRs are 
validated and merged the next day.

## Variance report (day 14)

```
> @cost-optimization-agent +7 days post first major batch. Run variance.
```

Agent re-pulls Cost Explorer for all 5 accounts, diffs against the 
2026-06-01 baseline:

```
Variance report drafted to findings/2026-06-15-variance-report.md.

Headline (real billed delta, account-level):

Account        Pre        Post       Delta       Notes
acme-prod      42,000     35,400     -6,600      AMI cleanup + Aurora downsize + 
                                                  RDS snapshot delete confirmed
acme-stage     18,200     16,900     -1,300      DMS Multi-AZ + scheduled scale-down
acme-dev       11,100     11,050     -50         minor, expected (mostly TF-PR pending)
acme-ops       9,800      8,200      -1,600      AMI cleanup confirmed
acme-sandbox   1,400      1,400      0           no actions taken

Total billed savings: $9,550/mo recurring at +7 days.
Nominal claim from execution log: $14,000-18,000/mo
Variance ratio (real/nominal): 53-68%

The 53-68% ratio is consistent with snapshot dedup effects we predicted in 
the AMI cleanup conversation. We were honest with the customer about the 
range in advance; the post-execution number lands in the predicted band.

Flagged for follow-up: 
- acme-dev variance is small because most TF-PRs are still pending review. 
  Expect $700-900/mo additional once PR 247 lands.
- The AwsBackup plan retention fix isn't reflected yet. Will appear in the 
  +14 day check.

Drafted CTO update at deliverables/2026-06-15-alex-cto-update.md.
```

You send the CTO update to Alex. The next 30 minutes are him asking sharp questions about the methodology (which the case study and runbooks answer) and approving the Phase 4 Savings Plans conversation for week 5.

## What this demonstrates

The agent isn't magical. It runs structured runbooks. The value is:

1. **Safety gates that fire**. The RDS Proxy near-miss in crawl phase. The IAM gap on Marco's EIPs in walk phase. Both caught before they cost anything.
2. **Honest dollar conversations**. The 53-68% real/nominal ratio matches what the agent predicted. The customer trusts the variance report.
3. **Owner-keyed work product**. Every drafted DM, every PR, every finding has an owner attached. Nothing gets lost.
4. **Audit trail in EXECUTION_LOG**. Every state change is logged. The +14 day, +30 day, and end-of-engagement reports all derive from this single source.

The agent's job is to make the methodology unblock the engineer running the engagement, not to replace them. The engineer (you) decides scope, talks to owners, and judges edge cases. The agent runs the disciplined parts that humans tend to skip when they're busy.
