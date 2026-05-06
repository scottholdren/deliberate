# Deliberate

> Issue-driven autonomous SDLC for Claude Code, with a planning gate that makes you think before agents start coding.

A Claude Code plugin that runs a disciplined multi-agent pipeline against your GitHub Issues. You decide what should be built; agents groom, architect, build, verify, review, secure, merge, and document. Human input is reserved for decisions that change the product — never for action approvals.

Built on top of [gstack](https://github.com/garrytan/gstack).

> **Status:** v0 — under active development. Not yet ready for general adoption.

## What it does

You start a project with `/init-project`. It runs a `gstack /office-hours` conversation with you, optionally puts the result through `/plan-ceo-review` and `/plan-eng-review`, then dispatches a ticket-groomer agent that decomposes the design into GitHub Issues plus an umbrella issue summarizing the proposed scope.

You review the umbrella, approve all of it or none of it, and close the umbrella. From that point on, agents drive: an architect plans each issue, a dev implements, qa/reviewer/security verify in parallel, a merger merges, a doc-writer updates docs. The pipeline runs in GitHub Actions; your dev environment is not involved.

When the system needs a decision from you — scope clarification, architectural fork, anything that would otherwise be a guess — it surfaces a well-researched question on the issue and `@`-mentions you. You answer in the email reply or as a comment, and the pipeline resumes.

You don't approve actions. You answer questions.

## Why this and not the other Claude Code SDLC plugins

The space is crowded but most existing plugins fall into two failure modes — sprawl (100+ agents, no opinions) or toy (handful of agents, no story for cost, retries, or failure handling). Deliberate is opinionated:

- Issue-driven, not chat-driven
- All-or-nothing planning gate (no drip-feeding)
- Pipeline-with-helpers composition (predictable state machine, not emergent delegation)
- Hard "judges can't write" boundary (reviewer/qa/security flag findings, never silently fix)
- Trend-aware retry budgets (extends on convergence, escalates on loops)
- Zero-config human channel (GitHub `@`-mention, delivered via email, replied via email)
- Cost-capped at every level
- Test what you ship

See [DESIGN.md](./DESIGN.md) for the full design, principles, and rationale.

## Install

> _TODO: install instructions once the plugin manifest is in place._

## Configuration

> _TODO: per-project config schema and examples._

## Architecture at a glance

```
Inception (local, you-driven)
  /init-project → gstack office-hours → ticket-groomer → umbrella issue + children

Human gate
  Approve all → all children become state:ready
  Or revise via comment → groomer reworks
  Or abandon → all closed as won't-do

Pipeline (cloud, GitHub Actions, per issue)
  architect → dev → [qa, reviewer, security in parallel] → merger → doc-writer

Operations (continuous)
  Scheduled sweeper handles stalls, dropped events, cost rollups, learnings sync
```

Full architecture: [DESIGN.md](./DESIGN.md).

## License

Apache-2.0. See [LICENSE](./LICENSE).

## Contributing

See [CONTRIBUTING.md](./CONTRIBUTING.md). The short version: we don't ship code we can't test. New notifier adapters, new agents, and new skills all require an end-to-end test before they merge.

## Security

Found a vulnerability? See [SECURITY.md](./SECURITY.md) for responsible disclosure.
