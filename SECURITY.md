# Security policy

Deliberate runs autonomously, holds API keys, and writes code into your repositories. Security matters.

## Reporting a vulnerability

Please **do not** open a public GitHub issue for security vulnerabilities.

Instead, email the maintainer or use GitHub's [private vulnerability reporting](https://docs.github.com/en/code-security/security-advisories/guidance-on-reporting-and-writing/privately-reporting-a-security-vulnerability) on this repository. Include:

- A description of the vulnerability
- Steps to reproduce
- Affected version(s)
- Proposed fix, if you have one

We aim to acknowledge within 72 hours and to release a fix within 14 days for high-severity issues.

## Trust boundaries

Deliberate operates across several trust boundaries. Treating them clearly is core to its design.

| Boundary | Trust assumption |
|---|---|
| The user (you) | Fully trusted |
| The GitHub repository | Trusted; commits represent your intent |
| GitHub Actions runners | Trusted execution environment; secrets must not leak to logs |
| Anthropic API | Trusted; calls go over TLS, prompts may include code |
| gstack skills | Trusted dependency; pinned to a known-good commit/tag |
| Third-party notifier endpoints (Discord, etc.) | Untrusted; we send only outbound webhooks, never accept inbound from them in v0 |
| LLM-generated code | Untrusted by default; passes through reviewer + qa + security agents and CI before merge |

## Secrets

- All secrets (`ANTHROPIC_API_KEY`, GitHub App credentials, notifier webhook URLs) live in GitHub Secrets, scoped per-repo or per-org. Never commit them.
- The plugin never logs secret values. Secrets are referenced in config by name, not by value.
- The bot's GitHub App tokens are short-lived installation tokens, not long-lived PATs.

## Permissions

- The Deliberate GitHub App requests the minimum permissions needed: issues r/w, pulls r/w, contents r/w, checks w. It does not request admin, billing, or org-level permissions.
- Branch protection on `main` is assumed and respected. The merger agent cannot bypass required reviews or status checks.
- Each subagent declares an explicit tool allowlist. Reviewer/QA/security agents have no Write or Edit tool access by design.

## Supply chain

- gstack is pinned to a specific commit or tag. Bumps are tested in CI before being released.
- Notifier adapters are tested end-to-end before being shipped. Adapters in `notifiers/CONTRIBUTING.md` are documented but not bundled.
- No transitive runtime dependencies fetched from the internet at agent execution time without lockfile pinning.

## What Deliberate will not do

- It will not exfiltrate code or credentials. The Anthropic API call is the only outbound destination for repository content; notifier webhooks receive only summaries with explicit URLs.
- It will not run arbitrary user-provided shell commands outside the dev-agent's scoped sandbox.
- It will not bypass branch protection.
- It will not auto-merge in v0.
- It will not silently install dependencies that are not already declared in your repo's manifest.

## Scope of this policy

This document covers the Deliberate plugin itself. It does not cover gstack, Claude Code, the Anthropic API, GitHub, or any third-party notifier service. For those, refer to their respective security policies.
