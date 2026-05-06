# Deliberate

> Issue-driven autonomous SDLC for Claude Code, with a planning gate that makes you think before agents start coding.

A Claude Code plugin that runs a disciplined multi-agent pipeline against your GitHub Issues. You decide what should be built; agents groom, architect, build, verify, review, secure, merge, and document. Human input is reserved for decisions that change the product — never for action approvals.

Built on top of [gstack](https://github.com/garrytan/gstack).

---

## 1. Why this exists

Claude can be trusted, with guardrails, to do the full SDLC: ticket grooming, PRs, deploys. The remaining bottleneck is the human dispatcher — the person who picks the next issue, prompts the next step, reviews the next PR, asks the next clarifying question.

Deliberate removes the dispatcher role. You set direction; agents drive. You stay in the loop only where your judgment is actually load-bearing: scope, taste, ambiguity in the spec, decisions that propagate. Everything else runs without you.

## 2. Positioning

The Claude Code SDLC orchestrator space is crowded. What makes Deliberate distinct:

- **Issue-driven, not chat-driven.** GitHub Issues are the input, the spec, and the durable record of every decision. You don't drive it from a terminal session.
- **All-or-nothing planning gate.** Before any code is written, the ticket-groomer creates an umbrella issue summarizing every proposed ticket. You approve all of it or none of it. Partial closure is not a state.
- **Pipeline-with-helpers composition.** A predictable state machine, not emergent delegation. Stages are state transitions; helpers are tool calls. Cost stays bounded, debugging stays tractable.
- **Hard "judges can't write" boundary.** Reviewer, QA, and security agents can flag findings but never silently fix code. This kills a class of self-marking-homework failure modes.
- **Human input only at decision points.** Never per-action approval. Decisions that shape the product surface as well-researched questions; everything else just happens.
- **Trend-aware retry budgets.** Static retry counts are wrong on both ends — too aggressive when converging, too lenient when looping. Deliberate extends the budget when finding counts decrease, escalates sooner when they don't.
- **Zero-config human channel.** Default communication is GitHub `@`-mentions on the canonical issue thread, which delivers via GitHub email and accepts replies via GitHub's reply-by-email feature. No webhook, no bot account linking.
- **Built on gstack.** Inherits a battle-tested set of skills (`/office-hours`, `/qa`, `/review`, `/cso`, `/ship`) rather than reimplementing them poorly.
- **Test what you ship.** No notifier adapter, no agent, no skill ships without an end-to-end test. Documentation describes more than the code implements only where the boundary is documented and stable.

## 3. How it works (high level)

```
Phase 0: Inception (local, you-driven)
  /init-project skill orchestrates:
    gstack /office-hours        → DESIGN.md
    gstack /plan-ceo-review     → revised DESIGN.md (optional)
    gstack /plan-eng-review     → revised DESIGN.md (optional)
    deliberate ticket-groomer   → child issues + umbrella issue

Phase 1: Human gate
  You review the umbrella issue:
    - Tick all boxes + close            → all children promoted to state:ready
    - Comment with feedback             → groomer revises, posts diff, asks for re-review
    - Apply decision:abandoned          → all children closed as won't-do
  Partial closure is not allowed.

Phase 2: Pipeline (cloud, GitHub Actions)
  For each state:ready issue:
    architect → dev → [qa, reviewer, security in parallel] → merger → doc-writer
  Verify findings flow back to dev as one consolidated comment per cycle.
  Agents that judge cannot write code; agents that produce code can.

Phase 3: Operations (continuous)
  Scheduled sweeper handles stalled issues, cost rollups, learnings sync,
  cross-project chores.
```

## 4. Architecture

### 4.1 Execution model

All orchestrator execution runs in GitHub Actions — both event-driven ticks and the scheduled sweeper. One execution environment, one auth model, one observability surface.

- **Event-driven workflow** (`.github/workflows/deliberate-events.yml`) listens for `issues.labeled`, `issues.closed`, `issue_comment.created`, `pull_request.opened`, `pull_request.synchronize`, etc. Each event invokes the orchestrator skill with a typed payload.
- **Scheduled sweeper** (`.github/workflows/deliberate-sweep.yml`) runs hourly. Detects stalled issues, dropped events, expired questions, cost-cap proximity. Idempotent.

The orchestrator is stateless between runs. State lives in the issue.

### 4.2 State management

State per issue lives in two layers:

- **Labels** carry the coarse state machine: `state:proposed`, `state:ready`, `state:planning`, `state:building`, `state:verifying`, `state:merging`, `state:documenting`, `state:done`, `blocked-on-human`, `decision:abandoned`. Plus `kind:issue-triage` for umbrella issues.
- **A pinned bot-owned comment per issue** carries the rich state: current step, last agent, cycle count, finding-count history (for trend-aware retries), running cost, links to verify reports, open questions. Edited in place each tick.

Umbrella issues are never written to after closure. They serve as the durable record of what was approved.

### 4.3 Inception flow

The `/init-project` skill is a local, conversational coordinator. It runs gstack skills in sequence — interleaved with human input where gstack expects it — then invokes the `ticket-groomer` agent to materialize issues from the resulting design document.

The umbrella issue uses GitHub's task-list syntax with one checkbox per child. Its body is owned by the agent and refreshed if the user requests revisions via comment. After closure (full approval), the body is frozen as a record.

### 4.4 Pipeline

Stages, in order, per issue:

| Stage | Owner agent | Output | Failure path |
|---|---|---|---|
| Plan | `architect` | Approach note posted on the issue | 1 redo allowed; second `wrong-direction` escalates |
| Build | `dev` | Branch + PR opened | Verify failures route back here |
| Verify | `qa`, `reviewer`, `security` (parallel) | Three full reports + one consolidated summary | Trend-aware retry, max 3-5 cycles |
| Merge | `merger` | PR merged, branch deleted | Manual escalation if merge conflicts persist |
| Document | `doc-writer` | Doc updates committed | Skipped if no doc impact detected |

Verify runs all three judges in parallel against the same PR commit. Findings are posted as three separate comments (durable, attributable), then consolidated into one summary comment with attribution and links back. Dev consumes the summary; the originals remain available for context.

### 4.5 Composition: pipeline-with-helpers

Stages are state transitions owned by the orchestrator. Helpers are synchronous tool calls available within a stage.

- An agent in a stage may call helpers from a curated allowlist; the helper returns, the parent agent continues, the stage advances normally.
- Helpers do not advance pipeline state on their own.
- Each helper has its own per-call budget.

This preserves a tractable macro state machine while accommodating natural cross-cutting concerns (e.g., the architect consulting security mid-design without needing a dedicated security stage).

### 4.6 Agent roster

**Pipeline agents** (one owns each stage):

| Agent | Stage | Tools |
|---|---|---|
| `ticket-groomer` | Inception → umbrella | GitHub Issues (CRUD), Read |
| `architect` | Plan | Read, GitHub (comment), helpers |
| `dev` | Build | Read, Write, Edit, Bash (scoped), GitHub (PR) |
| `qa` | Verify (parallel) | Read, Bash, GitHub (comment) — **no Write/Edit** |
| `reviewer` | Verify (parallel) | Read, Bash (`gstack /review`), GitHub PR review API — **no Write/Edit** |
| `security` | Verify (parallel) | Read, Bash (`gstack /cso`), GitHub (comment) — **no Write/Edit** |
| `merger` | Merge | GitHub (merge) only |
| `doc-writer` | Document | Read, Write, Edit (docs path only — enforced by hook), GitHub (commit) |

**Helpers** (callable as tools from any pipeline agent, allowlisted per agent):

- `codex-consult` — gstack `/codex consult` second opinion
- `security-consult` — quick security read, lighter than the full security stage
- `design-consult` — UI/UX take, gstack `/plan-design-review`
- `cost-check` — current spend on this issue vs. budget
- `human-question` — write a well-researched question to issue + notifiers, return to caller in a "blocked-on-human" state

Agents live as Claude Code subagents under `agents/` in the plugin. Each has frontmatter declaring its tool allowlist. No Agent SDK programs in v0.

### 4.7 Notifier abstraction

Notifications are decoupled from the orchestrator via an interface:

```
notify({
  title: string,
  body: markdown,
  action_url: string,
  severity: 'info' | 'action-required' | 'alert',
})
```

Adapters implement this contract and route to a delivery channel.

**v0 ships:**
- `github-mention` — default, zero config. The orchestrator `@`-mentions the relevant human in the canonical issue/PR comment. GitHub email handles delivery; reply-by-email handles answers.
- `discord-webhook` — opt-in. POSTs to a Discord webhook for severities the user configures.
- `console` — always-on fallback for debugging.

**Documented but not shipped:**
- `slack-webhook`, `teams-webhook`, `email-smtp`, generic `http` — interface is specified in `notifiers/CONTRIBUTING.md` with a stub template. Contributions welcome; we do not ship adapters we do not run.

Per-project configuration in `.deliberate/config.yml`:

```yaml
notifications:
  - kind: github-mention   # default, can be omitted
  - kind: discord-webhook
    webhook_url_secret: DISCORD_WEBHOOK_URL
    severities: [action-required, alert]
```

### 4.8 Notification rule

> **Documentation = thorough, in GitHub. Notification = only when human action is required.**

- Notify when: a question awaits, an umbrella is ready for review, the circuit breaker tripped, the cost cap is approaching, a PR is ready for human merge (if auto-merge is disabled).
- Stay silent for: PR opened, QA passed, reviewer approved, doc-writer pushed, agent completed. All observable in GitHub for the curious; never announced.
- `@`-mention the human only when a comment requires action.

## 5. Failure handling

| Concern | Default |
|---|---|
| Verify retry budget | Trend-aware: default 3 cycles; auto-extend to 5 if finding counts strictly decrease and no category recurs; hard cap 5. |
| Architect redo | 1 allowed. Second `wrong-direction` from reviewer escalates to a question. |
| Stalled in-progress | 2 hours with no progress → sweeper investigates. 7 days → human question. |
| Question unanswered | One reminder at 6 hours. Silent thereafter. Indefinite blocking is acceptable when advised. |
| Crash recovery | Sweeper detects "in-progress with no progress for >1hr and no clean exit" → resets the stage and re-invokes. Every agent must be idempotent. |
| Cost cap (per-issue) | Pause + question with cost breakdown and "extend / abandon / take over" choice. |
| Cost cap (per-day) | All in-flight issues pause until next day or manual unpause. |
| Bad PR detection | Reviewer can return `verdict:wrong-direction` instead of line-level nits. Routes back to architect, not dev. |
| Hard kill | Label `decision:abandoned` on any issue at any time → orchestrator stops touching it. Branch and PR left for manual cleanup. |

## 6. Cost controls

- **Per-task cap**: $5 default, overridable per issue via a label like `budget:25`.
- **Per-project per-day**: $50.
- **All-projects per-day**: $100.
- **Circuit breaker**: hard kill at 2x daily cap. Requires manual reset.
- **Telemetry**: cost logged per agent invocation, keyed to issue number. Daily/weekly rollups available via the sweeper.

## 7. Security & permissions

- **Bot identity**: a published GitHub App, installed per-org/per-repo. Permissions: issues r/w, pulls r/w, contents r/w, checks w. Short-lived installation tokens.
- **Secrets**: live in GitHub Secrets per repo. Never echoed to logs. Notifier webhook URLs referenced by secret name in config, never inlined.
- **Branch protection assumed**: `main` requires PR review and status checks. Deliberate respects branch protection; `merger` cannot bypass it.
- **Action permissions**: minimum necessary, declared per-workflow.
- **Audit trail**: every state transition is a comment edit or new comment. The pinned-state-comment's edit history is the audit log.
- **Trust boundary**: the orchestrator runs in GitHub-hosted runners by default. Self-hosted runners (e.g., proxmox home lab) are supported as an opt-in for cost control.

## 8. Project structure

```
deliberate/
├── plugin.json                       # Claude Code plugin manifest
├── README.md
├── DESIGN.md                         # this file
├── CHANGELOG.md
├── LICENSE                           # Apache-2.0
├── SECURITY.md
├── CONTRIBUTING.md
├── agents/                           # subagent definitions
│   ├── ticket-groomer.md
│   ├── architect.md
│   ├── dev.md
│   ├── qa.md
│   ├── reviewer.md
│   ├── security.md
│   ├── merger.md
│   └── doc-writer.md
├── helpers/                          # subagents called as tools
│   ├── codex-consult.md
│   ├── security-consult.md
│   ├── design-consult.md
│   ├── cost-check.md
│   └── human-question.md
├── skills/                           # top-level slash commands
│   ├── init-project/
│   │   └── SKILL.md
│   ├── orchestrator/
│   │   └── SKILL.md
│   └── status/
│       └── SKILL.md
├── notifiers/
│   ├── github-mention.ts
│   ├── discord-webhook.ts
│   ├── console.ts
│   └── CONTRIBUTING.md               # contract for new adapters
├── workflows/                        # GitHub Actions templates
│   ├── deliberate-events.yml
│   └── deliberate-sweep.yml
├── lib/                              # shared logic
│   ├── state.ts                      # pinned-comment parser/writer
│   ├── trend-budget.ts
│   ├── umbrella.ts
│   └── cost.ts
├── tests/
│   ├── fixtures/                     # toy repos for end-to-end tests
│   ├── e2e/                          # full pipeline runs
│   └── unit/
└── docs/
    ├── install.md
    ├── extending-notifiers.md
    └── self-hosted-runners.md
```

## 8.1 Gstack integration

Gstack is installed as a sibling of Deliberate, not a sub-component. Both live as Claude Code skill bundles in the same environment; agents reference gstack skills by their canonical prefixed names (`/gstack-office-hours`, `/gstack-qa`, `/gstack-review`, `/gstack-cso`, `/gstack-codex`, `/gstack-plan-design-review`, `/gstack-document-release`).

**Local install** (for the inception flow):
```
git clone --depth 1 https://github.com/garrytan/gstack.git ~/.claude/skills/gstack
cd ~/.claude/skills/gstack && ./setup
```

**Cloud install** (in GitHub Actions runners): the workflow templates clone gstack into `$HOME/.claude/skills/gstack/` before invoking `claude-code-action`. The runner is ephemeral so this happens fresh on every event tick — small cost, fully reproducible.

**Pinning**: the workflow templates default to `main` for v0.1. Before any tagged release, the templates pin `GSTACK_REF` to a known-good commit SHA. Bumps are tested against the e2e fixture before being released.

**Why a hard dependency, not vendored**: gstack moves fast and is widely used. Vendoring would fork its lineage, miss security fixes, and confuse contributors. Pinning is the right control.

**When gstack is unavailable**: agents that reference gstack skills are designed to degrade — they note "gstack-X not available, proceeding without" in their report and continue with their own logic. The pipeline does not hard-fail. (This is the current intent; degradation paths are tested as part of the e2e suite when those land.)

## 9. Distribution

- **License**: Apache-2.0.
- **Plugin**: published to the Claude Code plugin marketplace once stable.
- **GitHub App**: published as `deliberate-bot` (working name); users install per-org or per-repo.
- **Versioning**: SemVer from v0.1.0. CHANGELOG maintained from day one.
- **Releases**: tagged on the repo; the plugin manifest pins to a release tag, not `main`.
- **gstack pinning**: the plugin pins gstack to a known-good commit or tag. Bumps tested in CI.

## 10. Roadmap (rough)

**v0.1** — Scaffolding and one happy path
- Plugin manifest, agent skeletons, GitHub App registration docs
- `/init-project` runs gstack skills + ticket-groomer
- All-or-nothing umbrella mechanics
- Event workflow handles `state:ready` → architect → dev → verify → merger → doc-writer
- `github-mention` notifier
- One end-to-end test against a fixture repo
- Basic cost telemetry

**v0.2** — Reliability and observability
- Sweeper workflow with stall detection, dropped-event recovery, cost rollups
- Trend-aware retry budget
- Pinned-state-comment hardened against concurrent edits
- `discord-webhook` notifier
- Self-hosted runner documentation

**v0.3** — Quality of life
- `/status` skill for on-demand reports
- Per-project memory + global learnings PR proposals
- Cross-project cost dashboard

**v1.0** — Stable surface
- Public marketplace listing
- Documented extension points
- Used in anger across multiple real projects

## 11. Out of scope (won't build)

- Notifier adapters we do not personally run (Slack, Teams, email-SMTP, generic HTTP). The contract is documented; contributions welcome.
- Web dashboards or local servers. The state lives in GitHub; that is the dashboard.
- Discord 2-way conversation bridge. Email is the default 2-way channel; Discord stays outbound until there is a real testing story for inbound.
- Auto-merge in v0. The merger agent prepares the merge but a human approves until trust is established.
- Monorepo-aware orchestration in v0. One repo per orchestrator instance; monorepo support deferred.
- LLM-as-judge of LLM outputs beyond what gstack already does. We do not invent new evaluation systems.

## 12. Open questions (answer in implementation, not in chat)

- Exact tool allowlists per agent, especially the dev-agent bash sandbox boundary.
- Concurrency policy: how many issues advance in parallel per project before we hit rate limits.
- Idempotency keys for agent invocations to handle webhook retries.
- The exact trend-detection function for retry budgets (linear decrease vs. monotonic non-increase).
- Where global learnings memory lives and how it proposes plugin-level improvements as PRs.

## 13. Principles

In rough order of importance:

1. **The state of the repo is the truth.** Comments, labels, branches, PRs. No external dashboard.
2. **Don't ship what you can't test.** Adapters, agents, helpers, skills — all gated by an end-to-end test.
3. **Judges can't write.** Reviewer, QA, security report; they don't fix.
4. **Notify only for action.** Documentation is loud; notifications are quiet.
5. **Surface decisions, not approvals.** Questions cost the user judgment; routine actions cost the user nothing.
6. **Pipeline over delegation.** Predictability over emergent cleverness.
7. **Cloud-side execution.** Agents run where they can run reliably; humans code where they always have.
8. **Hard dependencies are honest.** We depend on gstack, we say so, we pin it.
9. **Names attribute, summaries link.** Every consolidated finding traces back to its source.
10. **Trust is earned per phase.** Auto-merge starts off; trust extends as the system proves itself.
