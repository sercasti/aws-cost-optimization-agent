# AWS Cost Optimization Agent

I have years of experience helping SaaS and platform teams cut 10-20% of their AWS bill in the first month, with the discipline to not break production while doing it. This repo is the methodology, runbooks, and Claude Code agent I use on those engagements. MIT-licensed. Take it, fork it, or ping me if you need help with this https://sercasti.github.io/.

## Recent result

A B2B SaaS customer running ~$102K/month across five AWS accounts (two Terraform-managed, three ClickOps).

**Five working days. ~$11K-$13K/month recurring savings captured. Another $5K-$15K/month identified and queued.**

| Outcome | Approximate $/mo |
|---|---|
| Quick Wins across 4 accounts (snapshots, EIPs, NAT GWs, DevOps Guru, gp2 to gp3, log retention) | ~$9,400 |
| Cross-account orphan sweep on day 5 (snapshots + 222 backing snapshots deleted) | ~$120 |
| AMI accumulation cleanup (104 AMIs deregistered, 89 TB nominal storage freed) | ~$1,400-3,200 |
| Stopped redundant RDS instance | ~$140 |
| Cross-account hygiene (stale alarms, idle resources, dereg orphans) | ~$50 |
| **Total realized week 1** | **~$11,100-$13,000** |
| Draft PRs in flight (Aurora downsize, DMS Multi-AZ off) | ~$1,200 |
| AMI follow-up + DMS rightsize + Aurora downsize queued | ~$3,000-9,000 |

Original portfolio: ~$102K/month. **Reduction: 11-13% in 5 days. Total opportunity within reach: 16-20%.**

Two near-misses caught at the safety gate, no PR opened on either:
1. An RDS Proxy "deletion candidate" claim that would have broken the production OpenFGA authorization service. Caught by mandatory pre-investigation grep.
2. A VPC endpoint "orphan" claim that would have hit `InvalidParameter: requester-managed` at the API. Caught by SG ownership check.

Both are documented in `docs/failure-modes.md`. The methodology is more credible when the war stories are included.

## How this runs

Four phases. Different risk profile and output format for each. Detail in `docs/methodology.md`.

1. **Crawl** (week 1, read-only): Cost Explorer baseline, per-account inventory, open-question intake.
2. **Walk** (weeks 1-2): direct-CLI Quick Wins on AWS-state items (snapshots, EIPs, log retention, etc.). Safety patterns applied.
3. **Run** (weeks 2-5): Terraform PRs for rightsizing, retention, scheduled scaling. Draft-PR pattern when local plan isn't feasible.
4. **Strategy** (week 4+): Savings Plans / Reserved Instance analysis. CTO-level conversation.

7 days after the bulk of Phase 2 lands, produce a variance report that diffs Cost Explorer line items against the engagement baseline. That's the artifact that turns "we did things" into "we saved you $X recurring."

## What's in this repo

```
.
├── docs/
│   ├── methodology.md         The 4-phase engagement model
│   ├── safety-patterns.md     Four reusable patterns for irreversible operations
│   └── failure-modes.md       Catalog of real-world gotchas with the fix for each
│
├── runbooks/                  Per-scenario playbooks
│   ├── orphan-snapshot-sweep.md
│   ├── unused-ami-audit.md
│   ├── cross-account-hygiene-sweep.md
│   ├── vpc-endpoint-classification.md
│   ├── rds-proxy-deletion-verification.md
│   └── tf-pr-draft-pattern.md
│
├── agents/
│   └── cost-optimization-agent.md   Claude Code subagent definition (system prompt + tools)
│
├── case-studies/
│   └── multi-account-saas-platform.md   The engagement that built this repo
│
├── DEMO.md                    What a real Claude Code session with this agent looks like
└── CONTRIBUTING.md            How to extend the runbooks
```

The repo works two ways:

**As reference**: read the methodology, runbooks, and failure modes. Take the patterns. Apply them yourself.

**As a working agent**: drop `agents/cost-optimization-agent.md` into your Claude Code project's `.claude/agents/` directory. The agent reads its instructions from this repo's docs and runs engagements with the safety patterns built in. See `DEMO.md` for a transcript of what that looks like end to end.

## Why this approach works

Cost optimization in AWS is mostly not about clever rightsizings. It's about removing waste without breaking production. Most engagements stall on the same five problems:

1. **False positives**. "This resource looks idle" frequently means "this audit missed a consumer." Lambda env vars, ECS task defs, SSM, Secrets Manager. The runbooks check all four before claiming anything is deletable.
2. **Owner approval vs. IAM grant**. Pre-approval is not the same as your user having the permission. Treat them as separate prerequisites.
3. **Auto-apply-on-merge pipelines**. Standard PR workflow ("plan before merge") is impossible when the maintainer is the only person who can plan. The TF-PR draft pattern handles this.
4. **Snapshot dedup**. A scan reporting "200 TB orphan storage" might bill for 40 TB after AWS block dedup. Real savings are 30-50% of nominal. Quote ranges, not point estimates.
5. **Recurring backups without retention**. AWS Backup will happily take a daily 1.4 TB AMI forever if the plan rule doesn't specify retention. Fix the policy, not just the symptoms.

Each of the five has a corresponding runbook or failure-mode entry in this repo.

## Need help with this?

https://sercasti.github.io/

## License

MIT. Use it, fork it, adapt it. If you ship something derived from this and want to credit, a backlink is appreciated but not required.

If you're a consultant running similar engagements and want to compare notes on what's worked and what hasn't, drop me an email. The methodology gets better with cross-engagement experience.
