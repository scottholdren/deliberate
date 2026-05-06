# Changelog

All notable changes to Deliberate are documented here. Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), versioning follows [SemVer](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- Initial design document (`DESIGN.md`) capturing the architecture, principles, and roadmap.
- Project scaffolding: `LICENSE` (Apache-2.0), `README.md`, `SECURITY.md`, `CONTRIBUTING.md`, `.gitignore`.
- Plugin manifest (`.claude-plugin/plugin.json`).
- Eight pipeline agents (`ticket-groomer`, `architect`, `dev`, `qa`, `reviewer`, `security`, `merger`, `doc-writer`) with role-scoped tool allowlists. Reviewer/QA/security have no Write or Edit access by design.
- Five helper agents (`codex-consult`, `security-consult`, `design-consult`, `cost-check`, `human-question`) callable from pipeline agents as tools.
- Three skills: `init-project` (local inception flow), `orchestrator` (event-driven dispatcher), `status` (on-demand snapshot).
- GitHub Actions workflow templates for event-driven orchestration and the hourly sweeper, under `templates/workflows/`.
- Notifier contract documentation (`notifiers/CONTRIBUTING.md`) for adapter authors.
- Plugin self-CI workflow (`.github/workflows/ci.yml`) validating manifest, agent frontmatter, skill frontmatter, and workflow YAML.
- Test scaffolding (`tests/README.md`).

### Changed
- Agent and skill files reference gstack skills by their `gstack-*` prefixed names (e.g., `/gstack-office-hours`). Subagent frontmatter declares required gstack skills via `skills:`.
- Workflow templates install gstack into `$HOME/.claude/skills/gstack/` and run `./setup --prefix` before invoking the orchestrator. Pinning is a TODO before v0.1 release.
- Removed incorrect `Bash(gstack *)` allowed-tool entries — gstack is a skill bundle, not a CLI binary.
- README install section documents the gstack `--prefix` requirement and per-repo workflow setup.

### Added (2)
- `DESIGN.md` section 8.1 documenting the gstack integration model, including the `--prefix` requirement and the rationale (gstack's own README recommends `--prefix` when running another skill pack alongside it).
- `docs/install.md` — step-by-step setup walkthrough covering GitHub App creation, secret configuration, gstack and Deliberate install, workflow templates, label seeding, and an end-to-end smoke test against a fresh test repo.
- `.claude-plugin/marketplace.json` — declares the Deliberate repo as a single-plugin marketplace so adopters can `/plugin marketplace add scottholdren/deliberate` and `/plugin install deliberate@deliberate`.

[Unreleased]: https://github.com/scottholdren/deliberate/compare/HEAD...HEAD
