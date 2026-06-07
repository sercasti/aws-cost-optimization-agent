# Contributing

This repo welcomes contributions: new runbooks, new failure-mode entries, refinements to the safety patterns, additional case studies (anonymized), or improvements to the agent definition.

## Before opening a PR

1. Run through the existing methodology in `docs/methodology.md`. Make sure your contribution fits the 4-phase model (Crawl / Walk / Run / Strategy) or, if it crosses phases, explicitly call out where it sits.
2. Check `docs/failure-modes.md` for adjacent gotchas. If your runbook would interact with a known failure mode, reference it explicitly.
3. Read `docs/safety-patterns.md`. Any runbook that recommends a destructive action must apply at least one safety pattern.

## What makes a good contribution

### Runbooks

A new runbook should:

- **Solve a specific scenario** ("Idle Redshift cluster cleanup," not "Redshift optimization in general").
- **Include realistic dollar ranges**. "Typical capture" sized from actual engagement experience, with caveats where they matter (snapshot dedup, reserved capacity remaining, etc.).
- **Include a Phase 1 read-only scan** that the user can run safely before any destructive action.
- **Include classification rules** (auto-eligible vs. owner-confirmation vs. never).
- **End with an honest "what this captured in practice" section** that shows real numbers and any near-misses.
- **No em dashes anywhere**. Use colons, periods, parentheses, or hyphens.

Template:

```markdown
# Runbook: <scenario>

**Goal**: <one sentence>
**Typical capture**: $X-Y/mo for <profile>
**Risk profile**: low | medium | high

## Why this exists (optional, only if methodology is non-obvious)

## Phase 1: Scan (read-only)
<commands + outputs>

## Phase 2: Classify
<rules>

## Phase 3: Owner conversation (if needed)
<template>

## Phase 4: Execute
<commands + safety pattern applied>

## Verification
<how to confirm in Cost Explorer>

## What this captured in practice
<real numbers from engagement>
```

### Failure mode entries

A new entry in `docs/failure-modes.md` should:

- **Have actually happened**, either in your engagement work or in a published incident. Speculative gotchas are less useful than confirmed ones.
- Include: symptom, root cause, fix, lesson. All four. Skipping "lesson" makes the entry hard to internalize.
- Stay short. The catalog is more useful at 20 entries than at 100 verbose ones.

### Safety patterns

New patterns are rare; the existing four cover most cases. If you propose a fifth:

- Explain what class of mistake it prevents that the existing patterns don't.
- Include a code example.
- Include the "what's the cost during the safety window" calculation.

### Case studies

Anonymize aggressively. The customer must not be identifiable from:

- Industry-specific resource naming patterns.
- Person names, role titles, or role-+-account combinations.
- Specific dollar amounts (use ranges).
- Combinations of services (e.g., "ECS + Aurora + OpenFGA" might point to a specific customer; describe as "containerized API behind a relational backend").

When in doubt, leave the detail out. The methodology is more transferable when it's generic.

## Style

- **No em dashes**. Use colons, periods, parentheses, or regular hyphens. This is a hard rule.
- **Argentinian Spanish for owner-facing templates** when the customer's working language is Spanish. Plain English otherwise.
- **Concrete over abstract**. "31 orphan RDS snapshots, ~$293/mo" beats "significant savings."
- **Honest about what didn't work**. The repo is more credible when failure modes are documented than when only successes are.

## Testing

This repo is documentation + agent definitions, not executable code. "Testing" means:

- **Runbook commands work** when copy-pasted into a real AWS CLI session. If you can run them against a sandbox account and they execute cleanly, that's the bar.
- **Agent definition loads** in Claude Code. Drop it into `.claude/agents/` and invoke it. Confirm the agent reads its inputs (this repo's docs) before acting.
- **Markdown renders** correctly on GitHub. Headers, code blocks, tables.

## PR review process

PRs are reviewed by the repo maintainer. Expect 1-3 business days for an initial pass. Substantial contributions (a new runbook, a new case study) may take longer because they get more scrutiny on tone, anonymization, and methodology fit.

## What I won't merge

- Speculative methodology without engagement experience behind it.
- Case studies where anonymization is incomplete.
- Runbooks that recommend destructive action without a safety pattern.
- Em dashes.

## What I will merge eagerly

- New runbooks for AWS services not yet covered (DocumentDB, Neptune, MWAA, OpenSearch, etc.) with the template above.
- New failure mode entries with the four-part structure.
- Improvements to the agent definition that catch a class of mistake the existing version misses.
- Spanish-language owner DM templates that match the existing tone.

## Contact

Open an issue or reach out via the contact info in the README.
