# Install & first run

This doc walks you from zero to a working Deliberate setup against a test repo. Follow in order. Once you complete the smoke test at the end, you'll have seen the orchestrator drive a single pipeline tick end-to-end.

> **Time estimate**: ~30 minutes for the full setup, most of which is GitHub App configuration.

## Prerequisites

- GitHub account with admin access to (or ability to create) a test repo.
- `gh` CLI installed and authenticated (`gh auth status` to verify).
- `git` installed.
- Anthropic API key (https://console.anthropic.com).
- Claude Code installed locally (only needed if you want to run `/init-project` locally; the orchestrator itself runs in cloud Actions).

## Step 1: Create a GitHub App

1. Go to https://github.com/settings/apps/new (personal) or `https://github.com/organizations/<org>/settings/apps/new` (org).

2. Fill in the form:
   - **GitHub App name**: `deliberate-bot-<your-username>` (must be globally unique on GitHub).
   - **Description**: optional, but useful for your future self. Suggested: `Deliberate orchestrator bot — drives the SDLC pipeline against issues in this repo. https://github.com/scottholdren/deliberate`
   - **Homepage URL**: your fork of the Deliberate repo URL is fine.
   - **Webhook**: **Uncheck "Active"**. Deliberate uses GitHub Actions for event delivery, not the App's webhook. With "Active" unchecked, the URL and secret fields are ignored and the "Subscribe to events" section is hidden (which is what we want — leave it hidden).
   - **Repository permissions** (alphabetical, set to the value indicated):
     - Checks: **Read and write**
     - Contents: **Read and write**
     - Issues: **Read and write**
     - Metadata: **Read-only** (auto-selected, can't be changed)
     - Pull requests: **Read and write**
   - **Where can this GitHub App be installed?**: "Only on this account" is fine for a personal test.

3. Click **Create GitHub App**.

4. On the next page, note the **App ID** (a numeric value at the top). You'll need it.

## Step 2: Generate a private key

1. On the App's settings page, scroll to **Private keys**.
2. Click **Generate a private key**. A `.pem` file downloads.
3. Keep this file safe — it's the App's credential. You'll paste its contents into a GitHub Secret in step 4.

## Step 3: Install the App on a test repo

1. Create a fresh test repo (or pick one you don't mind experimenting in). One command does creation + initial README + clone:
   ```
   gh repo create deliberate-fixture --public --add-readme --clone \
     --description "Deliberate test repo"
   cd deliberate-fixture
   ```

   You should land in a working clone with one commit on `main` (the auto-generated README).

2. On your App's settings page, click **Install App** in the left sidebar.

3. Click **Install** next to your account.

4. Choose **Only select repositories**, pick `deliberate-fixture`, and click **Install**.

The App is now authorized on that repo. The Actions runner will be able to authenticate as it.

## Step 4: Configure repo secrets

In the **`deliberate-fixture`** repo (not the Deliberate repo):

1. Go to **Settings** → **Secrets and variables** → **Actions**.

2. Click **New repository secret** and add each of these:

   | Name | Value |
   |---|---|
   | `ANTHROPIC_API_KEY` | Your Anthropic API key |
   | `DELIBERATE_APP_ID` | The App ID from step 1.4 |
   | `DELIBERATE_APP_PRIVATE_KEY` | The full contents of the `.pem` file from step 2 (paste including `-----BEGIN RSA PRIVATE KEY-----` and `-----END RSA PRIVATE KEY-----`) |

   Optional, only if using Discord:
   | `DISCORD_WEBHOOK_URL` | A Discord incoming webhook URL |

3. Verify with `gh secret list --repo <your-username>/deliberate-fixture`. You should see the three required secrets.

## Step 5: Install gstack locally with `--prefix`

Deliberate references gstack skills by their `gstack-*` prefixed names. Install gstack with `--prefix`:

```
git clone --depth 1 https://github.com/garrytan/gstack.git ~/.claude/skills/gstack
cd ~/.claude/skills/gstack && ./setup --prefix
```

Verify: open Claude Code and run `/gstack-office-hours --help` (or just `/` and look for `gstack-*` entries in the autocomplete). If you see them, you're done.

If you previously installed gstack without `--prefix`, re-run with the flag. Existing prefix-less symlinks in `~/.claude/skills/` will be replaced.

## Step 6: Install Deliberate

In Claude Code, run:

```
/plugin marketplace add scottholdren/deliberate
/plugin install deliberate@deliberate
```

The first command registers the Deliberate repo as a single-plugin marketplace. The second installs the plugin. The `@deliberate` suffix is the marketplace name (derived from the repo name) — it is required, the install fails silently without it.

**Scope prompt**: the installer asks where to enable the plugin:
- **User** — enabled for all projects on this machine. Persists across sessions. **Recommended** for the project owner; you'll want Deliberate available wherever you run Claude Code.
- **Project** — enabled only for the current project (committed to `.claude/settings.json`).
- **Local** — enabled only for the current project, for you only (in `.claude/settings.local.json`, gitignored).

Pick **User**.

**Reload required**: after install, run:

```
/reload-plugins
```

The plugin's slash commands aren't available in the current session until you reload.

Verify: type `/` and look for `init-project`, `orchestrator`, and `status` in the autocomplete. `/plugin list` should also show `deliberate` enabled.

### Alternative: local-clone path for development

If you're hacking on Deliberate itself and want to test local changes without going through GitHub:

```
git clone https://github.com/scottholdren/deliberate.git ~/code/deliberate
claude --plugin-dir ~/code/deliberate
```

Then `/reload-plugins` inside the session after edits.

## Step 7: Copy workflow templates and seed labels

In `deliberate-fixture`:

1. Copy the workflow templates:
   ```
   mkdir -p .github/workflows
   curl -sL https://raw.githubusercontent.com/scottholdren/deliberate/main/templates/workflows/deliberate-events.yml \
     > .github/workflows/deliberate-events.yml
   curl -sL https://raw.githubusercontent.com/scottholdren/deliberate/main/templates/workflows/deliberate-sweep.yml \
     > .github/workflows/deliberate-sweep.yml

   git add .github/workflows
   git commit -m "Add Deliberate workflows"
   git push
   ```

2. Create the labels Deliberate uses:
   ```
   for label in \
     "state:proposed" \
     "state:ready" \
     "state:planning" \
     "state:building" \
     "state:verifying" \
     "state:merging" \
     "state:documenting" \
     "state:done" \
     "blocked-on-human" \
     "decision:abandoned" \
     "kind:issue-triage"; do
     gh label create "$label" --repo <your-username>/deliberate-fixture --color "ededed" || true
   done
   ```

   (`ticket-groomer` will also create these as needed during the inception flow, but for a manual smoke test we seed them now.)

## Step 8: Smoke test

This is the moment of truth. We'll create a single trivial issue, label it `state:ready`, and watch the orchestrator drive it through one pipeline tick.

1. Create a test issue:
   ```
   gh issue create --repo <your-username>/deliberate-fixture \
     --title "Add a Hello World file" \
     --body $'Add a file `hello.txt` at the repo root containing the text `Hello, world!`.\n\n## Acceptance criteria\n- File exists at `hello.txt`\n- Contents are exactly `Hello, world!`\n- A test verifies the file exists (use `test -f hello.txt` in CI or the project\'s test runner if one exists)'
   ```

2. Note the issue number in the output (`#1`, etc.).

3. Apply `state:ready`:
   ```
   gh issue edit <num> --repo <your-username>/deliberate-fixture --add-label state:ready
   ```

4. **Within ~30 seconds**, watch the workflow start. Open the Actions tab in the browser, or:
   ```
   gh run list --repo <your-username>/deliberate-fixture --workflow deliberate-events.yml --limit 5
   gh run watch --repo <your-username>/deliberate-fixture
   ```

5. **What you should see** on a successful first tick:
   - The workflow runs to completion (green check).
   - The issue gets a new `state:planning` label.
   - A new comment appears on the issue starting with `<!-- deliberate:state -->` — this is the pinned-state-comment.
   - A second comment appears titled "Implementation plan" — the architect's output.
   - You may also see the workflow re-trigger if the architect's plan posting causes a follow-on event; that's normal.

6. **What you should not see** (yet):
   - A PR. The dev agent runs in a subsequent tick. (We're testing one tick at a time for now.)

## What "success" means at this stage

If you reach step 8.5 with the architect plan posted, the entire chain works: GitHub App auth, gstack install in the runner, plugin loading, orchestrator dispatch, subagent invocation, GitHub state writes. That's the hard part. Everything else is incremental wiring.

If something fails, see Troubleshooting below — and please open an issue on the Deliberate repo with the workflow log if it's a real bug, since this doc is itself untested at this stage and we want to fix any rough edges fast.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Workflow doesn't trigger when label applied | Workflow file has YAML errors, or `state:ready` label doesn't exist yet | Check Actions tab for parse errors. Re-run the label creation script in step 7. |
| `Resource not accessible by integration` | App permissions don't include the operation being attempted | Re-check step 1.2 permissions. Reinstall the App if you change permissions (GitHub requires re-acceptance). |
| `Bad credentials` on the App token step | App ID or private key incorrect in secrets | Re-verify secret values. The private key must include the `BEGIN`/`END` lines. |
| `gstack: command not found` or skills missing | gstack install in the runner failed | Check the workflow log for the gstack install step. Confirm `GSTACK_REF` (defaults to `main`) is reachable. |
| Orchestrator runs but nothing changes on the issue | Probably a logic bug in the orchestrator skill | Open the workflow log, look at the Claude Code action's output. The orchestrator should print a summary of what it did. |
| Architect posts but no `state:planning` label | Race or label-update permission | Re-check Issues permission. Try re-applying `state:ready` to retrigger. |

## Cost expectations

A single tick (architect or dev) typically runs ~$0.10–$2 depending on repo size. The smoke test in step 8 is a single architect invocation against a tiny repo — expect under $0.50.

If you see a tick burning more than a few dollars, kill the workflow and check the orchestrator's logs for runaway loops.

## After the smoke test

Once step 8 works end-to-end, you can graduate to the full inception flow:

```
cd /path/to/deliberate-fixture
claude
> /init-project
```

This walks through office-hours, optional plan reviews, and ticket-grooming. The result: an umbrella issue you review and close to start the full pipeline against your own design.
