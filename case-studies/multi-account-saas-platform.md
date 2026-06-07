# Case study: multi-account SaaS platform, 5 working days, 11-13% portfolio reduction

A summary of the engagement that built the methodology and runbooks in this repo. All customer-identifying details are removed. Numbers are real.

## Customer profile

A B2B SaaS platform operating across five AWS accounts:

- 2 Terraform-managed accounts (mature IaC, auto-apply-on-merge CI).
- 3 ClickOps accounts (historical mix of Console-created and ad-hoc CLI resources).
- Combined AWS spend: ~$102K/month at engagement start.
- Stakeholders: CTO (engagement sponsor), 4 account owners (each owning 1-2 accounts).

## Scope of engagement

Originally framed as a 6-week structured optimization engagement. Phase 1 (Crawl) ran over the first 5 working days, alongside Phase 2 (Walk) executions on accounts where data and owner consent were both ready.

This case study covers the first 5 working days specifically.

## Day-by-day summary

### Day 1: Access setup, Cost Explorer baseline

- Configured CLI access for all 5 accounts (mix of IAM users via `aws login` and SSO via IAM Identity Center).
- Pulled Cost Explorer line-item breakdowns for each account, identifying the top 10 line items per account by monthly spend.
- Result: identified the cost categories that would matter (EBS snapshot storage, RDS Multi-AZ, NAT Gateway data transfer, idle DMS replication, oversized Aurora instances, unmanaged log retention).

### Days 2-3: Per-account scans + initial Quick Wins

Per the methodology, Phase 2 ran on the ClickOps accounts first because direct-CLI cleanup creates no Terraform drift there.

- **31 orphan RDS snapshots deleted** across one account (manual snapshots whose source DBs were long gone). ~$293/mo recurring.
- **Stopped EC2 instance cleanup** flagged for owner confirmation (5 instances, ~$220/mo aggregate, awaiting owner sign-off).
- **DevOps Guru disabled** on 2 accounts where it was billing without active review. ~$50/mo combined.
- **CloudWatch log retention** set on 30+ log groups across the portfolio. $0 immediate, bounds future cost.
- **gp2 → gp3 EBS storage migration** on 10 RDS instances (IOPS-throttled io2 ceiling moved to gp3 baseline). ~$2,083/mo savings, also resolved real production throttling caught during the scan.

**End of day 3**: ~$9,400/mo recurring captured from Quick Wins across ClickOps accounts.

### Day 4: TF-PR sequence opened

The Terraform-managed accounts came online via the draft-PR pattern:

- **Aurora downsize PR** opened in draft: dev/uat clusters from r7g.large to r7g.medium. ~$570/mo. Waiting on maintainer plan validation.
- **DMS Multi-AZ disable PR** opened in draft: production replication instance idle for 14+ days, Multi-AZ premium unjustified. ~$594/mo. Waiting on maintainer plan validation.
- **CloudWatch validations** posted as PR comments: DMS prod CPU at 0.42% avg (justifying potential rightsize on top of Multi-AZ off), Aurora dev/uat writers at 9-11% avg CPU and < 4 GB memory used (justifying downsize). Kinesis stream `*-zones-events` showed zero traffic over 30 days, strengthening a third PR opportunity.

Combined PR pipeline value: ~$1,164/mo immediately + ~$446/mo additional from the DMS rightsize follow-up = ~$1,610/mo in flight pending maintainer review.

### Day 5: Cross-account hygiene + the big AMI discovery

Two parallel workstreams.

**Hygiene sweep across 5 accounts**:

- 1 orphan VPC deleted (Terraform-managed account, pre-approved, no resources attached). Pure hygiene win.
- 1 orphan EBS volume (600 GB) found in another account, source snapshot already deleted. Snapshot + delete pattern applied to preserve recoverability. ~$60/mo after the 30-day safety hold.
- 6 RDS manual snapshots whose source DBs were long gone (multiple accounts). ~$115/mo.
- 7 EBS snapshots whose AMIs had been deregistered. ~$5/mo.
- 23 log groups bulk-set to 30-day retention. Bounds future cost.
- 15 CloudWatch alarms in INSUFFICIENT_DATA state since 2022-2024 (cleanup).

**The big AMI discovery**:

A targeted AMI audit across the two largest accounts revealed:

| Account | Unused AMIs | Nominal storage | Nominal $/mo |
|---|---|---|---|
| SaaS application account | 258 | 200 TB | ~$10,000 |
| IoT data account | 136 | 164 TB | ~$8,200 |
| Other 3 accounts | 4 | 500 GB | ~$25 |
| **Total** | **398** | **~365 TB** | **~$18,250** |

Pattern: ~80% was manual pre-maintenance backups (operators creating AMIs before risky changes and never deleting them, going back years). ~20% was AWS-Backup-managed daily AMIs of large instances. Real billed cost (after AWS snapshot block dedup): estimated $5,500-9,000/mo, but only Cost Explorer post-cleanup variance would give the actual number.

**Owner conversation**: presented the finding with the dedup caveat. Owner approved batch deletion of all unused AMIs older than 12 months. Executed:

- Batch 1: 81 AMIs, 168 backing snapshots, 67 TB nominal freed.
- Batch 2 (follow-up: "extend to anything before Jan 1"): 23 more AMIs, 54 snapshots, 22 TB nominal.
- Total: **104 AMIs, 222 backing snapshots, ~89 TB nominal storage freed in one afternoon.**

Real billed savings (+7 day Cost Explorer check): ~$1,400-$3,200/mo recurring.

## End-of-week tally

| Outcome | $/mo recurring |
|---|---|
| Initial Quick Wins (snapshots, EIPs, DevOps Guru, gp3 migration, log retention) | ~$9,400 |
| Cross-account orphan snapshot sweep | ~$120 |
| AMI accumulation cleanup | ~$1,400-3,200 |
| Stopped instance cleanup (1 captured, others pending owner) | ~$140 |
| Stale alarm + hygiene | ~$50 |
| **Realized in 5 working days** | **~$11,100-$13,000** |
| TF-PRs in flight | ~$1,164 |
| AMI follow-up + DMS rightsize + Aurora downsize queued | ~$3,000-9,000 |
| **Total opportunity within reach** | **$15,000-22,000** |

Original portfolio: ~$102K/mo. **Realized cost reduction: 11-13% in week 1. Identified-and-queued: another 5-15%.**

## What the customer got that wasn't dollars

Three things from this engagement that the customer flagged as more valuable than expected:

1. **Variance report methodology**. The customer's prior cost-savings consultants quoted nominal savings figures that didn't match Cost Explorer. The post-execution +7-day variance report from this engagement was the first time the CTO had savings claims they could trace to specific Cost Explorer line items.

2. **Two near-misses caught at the safety gate**. (a) The RDS Proxy "deletable" claim that would have broken OpenFGA authz. (b) The VPC endpoint "orphan" claim that would have hit `InvalidParameter: requester-managed` errors and surfaced a misclassification mid-execution. Both were caught in pre-investigation, no PR opened, no customer impact.

3. **A pattern fix at the source**. The AwsBackup retention conversation that came out of the AMI audit isn't a one-time cleanup. Setting the right retention on the backup plan prevents the accumulation pattern from recurring. The customer's previous practice was to manually clean every 6-12 months; the new policy auto-prunes.

## What didn't go to plan

In the interest of honesty:

1. **Initial RDS Proxy audit was wrong**. Claimed $2,466/mo of deletable RDS Proxy infrastructure. Pre-investigation grep revealed the proxies were load-bearing. The claim was retracted from the customer savings tracker before any PR was opened. Net dollar impact: $0. Net process impact: the failure-modes catalog grew by 2 entries (Windows MSYS bug, VPC endpoint requester-managed classification).

2. **Initial AMI cleanup approval was conservative (12 months)**. Owner's actual preference was "before Jan 1" (~6 months). Second batch had to be run as a follow-up. This was a missed opportunity for a single tighter ask but was easily corrected.

3. **Two account closures were blocked at IAM**. Operational pre-approval was confirmed; the API rejected with AccessDeniedException because `organizations:CloseAccount` wasn't on the user's policy. ~$8/mo of cleanup left on the table; the owner agreed to either run the commands themselves or grant the permission.

## Methodology takeaways the customer kept

After the engagement, the customer adopted these patterns internally:

1. Pre-delete safety snapshot tag scheme (`Purpose=pre-delete-safety`, `ReviewBy=<date>`).
2. The 4-source verification gate for any "this resource is unused" claim.
3. The TF-PR draft pattern for cost-savings PRs in their auto-apply repos.
4. Quarterly cross-account hygiene sweep (now on their roadmap).

The methodology lives in `docs/methodology.md` in this repo. The runbooks they execute live in `runbooks/`. Both are MIT-licensed.
