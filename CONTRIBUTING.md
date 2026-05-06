# Contributing to Deliberate

Thanks for your interest. Read this before opening a PR.

## Principles

These come from [DESIGN.md](./DESIGN.md) and shape what gets accepted:

1. **Don't ship what you can't test.** New agents, helpers, notifier adapters, and skills must come with an end-to-end test against a fixture repo. PRs that add functionality without tests will be asked to add them or be reduced to documentation.
2. **Judges can't write.** Reviewer, QA, and security agents must not have Write or Edit tool access. Don't loosen this in a PR without an explicit design discussion in an issue first.
3. **Pipeline over delegation.** Stages are state transitions; helpers are tool calls. Don't introduce emergent agent-to-agent invocation outside the curated helper allowlist.
4. **Notify only for action.** New notification points must require human action. Routine progress updates do not get notifications.
5. **The state of the repo is the truth.** No external dashboards, no parallel state stores, no ephemeral memory beyond what's recorded in issue labels and the pinned-state-comment.

## Before opening a PR

1. Open an issue first for anything beyond a small fix or doc tweak. The maintainers will respond with whether it fits Deliberate's scope.
2. Check the [out-of-scope list](./DESIGN.md#11-out-of-scope-wont-build). Slack/Teams/SMTP/HTTP notifiers, web dashboards, auto-merge, and monorepo support are explicitly deferred.
3. Read the relevant section of DESIGN.md so we're working from the same model.

## Development setup

> _TODO: setup instructions once the plugin scaffold is finalized._

## Running tests

> _TODO: test commands._

The end-to-end suite runs the full pipeline against fixture repos in `tests/fixtures/`. CI runs both unit and e2e on every PR.

## Adding a notifier adapter

The notifier contract is documented in [`notifiers/CONTRIBUTING.md`](./notifiers/CONTRIBUTING.md). The interface is four fields:

```ts
notify({
  title: string,
  body: string,    // markdown
  action_url: string,
  severity: 'info' | 'action-required' | 'alert',
})
```

Bundled adapters (currently `github-mention`, `discord-webhook`, `console`) live in `notifiers/`. Other adapters are welcome via the contract, but we do not bundle adapters we do not run in production. If you want to maintain a `slack-webhook` or `teams-webhook` adapter as a separate package that conforms to the contract, link it in the README.

## Adding an agent

A new agent must:

- Have a clear single-stage role in the pipeline, or be a helper that is callable as a tool
- Declare its tool allowlist in the frontmatter
- Be testable with an end-to-end scenario in `tests/e2e/`
- Have a corresponding rationale in DESIGN.md (open a PR to DESIGN.md alongside)

Agent sprawl is a known failure mode for Claude Code SDLC plugins. We will push back on agents that don't earn their keep.

## Style

- Markdown agent and skill files use frontmatter for metadata, plain prose for instructions. Don't write multi-paragraph prose where a checklist will do.
- Code (TypeScript, Bash, etc.): no comments unless the why is non-obvious. Identifiers should explain what; PRs should explain why.
- No emojis in shipped artifacts.

## License

By contributing, you agree your contributions will be licensed under Apache-2.0.
