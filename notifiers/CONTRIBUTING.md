# Notifier contract

Notifiers are how Deliberate tells you something needs your attention. The contract is intentionally tiny so that adding a new channel (Slack, Teams, Telegram, PagerDuty, anything HTTP) is straightforward.

## The interface

Every notifier implements one operation:

```ts
notify({
  title: string,           // one-line summary, max ~80 chars
  body: string,            // markdown, no length limit (but be reasonable)
  action_url: string,      // absolute URL the user clicks to act
  severity: 'info' | 'action-required' | 'alert',
})
```

That's the entire contract. Inputs are guaranteed to be sanitized markdown — no HTML, no shell metacharacters in the URL.

## What ships in v0

| Adapter | Status | Use |
|---|---|---|
| `github-mention` | Bundled, default | The orchestrator `@`-mentions the user in the canonical issue/PR comment. GitHub email handles delivery; reply-by-email is the answer path. Zero configuration required. |
| `discord-webhook` | Bundled, opt-in | Outbound only. POSTs to a Discord incoming webhook. Useful for mobile push or team visibility. |
| `console` | Bundled, fallback | Logs to GitHub Actions output. Always-on, useful for debugging or as a literal "no notifier configured" fallback. |

## What does not ship

- `slack-webhook`, `teams-webhook`, `email-smtp`, generic `http`: documented contract, no bundled implementation. We don't ship code we don't run in production. If you write one, see "Contributing your adapter" below.

## Configuration

Per project, in `.deliberate/config.yml`:

```yaml
notifications:
  - kind: github-mention   # default; can be omitted
  - kind: discord-webhook
    webhook_url_secret: DISCORD_WEBHOOK_URL   # name of a GitHub Secret, never the value
    severities: [action-required, alert]      # which severities fire this adapter
```

`webhook_url_secret` is resolved at run time from GitHub Secrets. The value is never logged.

## Severity rules

- `info` — rarely used. Reserve for opt-in cost rollups, weekly digests.
- `action-required` — the user must do something for the pipeline to proceed (answer a question, review an umbrella, decide on cost extension). This is the most common severity.
- `alert` — circuit breaker tripped, daily cost cap hit, anomaly that needs immediate attention. Loud channels (PagerDuty-class) only.

## Writing a new adapter

A notifier lives in a single file in `notifiers/<name>.ts` (or `.js`, `.py` — language conventions in the plugin will be settled in v0.2). Skeleton:

```ts
import type { NotifyInput } from "./types";

export async function notify(input: NotifyInput): Promise<void> {
  // Read your secret from process.env (the orchestrator passes it through
  // based on `webhook_url_secret`).
  const url = process.env.MY_ADAPTER_WEBHOOK_URL;
  if (!url) throw new Error("MY_ADAPTER_WEBHOOK_URL not set");

  // Translate input into your channel's native format.
  const payload = {
    text: `${input.title}\n\n${input.body}\n\n${input.action_url}`,
  };

  const res = await fetch(url, {
    method: "POST",
    headers: { "content-type": "application/json" },
    body: JSON.stringify(payload),
  });
  if (!res.ok) throw new Error(`notifier failed: ${res.status}`);
}
```

## Testing requirements

Per the [`Don't ship what you can't test`](../CONTRIBUTING.md) principle:

- An adapter PR must include an end-to-end test that posts a real notification to a real (test) endpoint.
- The test must run in CI on every PR.
- Adapters without a test path are not bundled — but the project welcomes a link in the README to a separately-maintained adapter package that conforms to the contract.

## Contributing your adapter

If you maintain an adapter for a channel we don't bundle:

1. Publish it as a separate package or repo.
2. Confirm it conforms to the contract above.
3. Open a PR adding a "Compatible adapters" section to the project README with a link.

We do not bundle adapters we don't run, but we do happily list third-party adapters that conform.
