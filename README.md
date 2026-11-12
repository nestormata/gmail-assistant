# gmail-cleaner

Multi-account Gmail triage toolkit. Connects to Google Mail accounts **one at a
time**, mirrors message **metadata** into a local SQLite database, applies
deterministic rules, and escalates only *unknown senders* to a local LLM
(Qwen2.5-Coder-7B on the Windows box) for classification. Bulk changes go
through Gmail's batch endpoints. **Dry-run is the default for everything that
modifies the mailbox.**

## Design

```
sync ──► SQLite mirror ──► rules (deterministic) ──► plan
                                   │ unknown senders only
                                   ▼
                          LLM sender classification  (commandcenter:8080)
                                   ▼
                              plan ──► apply (--apply required)
```

- The model classifies **senders**, not messages: one decision covers every
  past and future email from that sender, and prompts stay tiny (fits the 8k
  context of the 7B model).
- Rules matched senders never reach the model. `protect:` entries in
  `accounts.yaml` are checked first and are never modified by anything.
- `gmail.modify` scope cannot hard-delete — only trash (recoverable 30 days).
  Hard delete is intentionally unsupported.
- Sending requires `allow_send: true` on the account **and** `--yes` on the
  command.

## One-time Google Cloud setup (manual, browser needed)

1. <https://console.cloud.google.com> → create project (e.g. `gmail-cleaner`).
2. **APIs & Services → Library** → enable **Gmail API**.
3. **APIs & Services → OAuth consent screen**:
   - User type: **External** (consumer Gmail) or **Internal** (Workspace).
   - **Publishing status: In Production** — this is the step that stops
     refresh tokens expiring every 7 days (Testing mode revokes them).
     Unverified production is fine for personal use (<100 users): each
     account consents once through an "unverified app → Advanced → continue"
     screen.
   - Add scopes:
     `https://www.googleapis.com/auth/gmail.modify`
     `https://www.googleapis.com/auth/gmail.send`
4. **Credentials → Create credentials → OAuth client ID → Desktop app** →
   download the JSON → save as `config/client_secret.json`
   (gitignored; the tools load it, nobody should read it into chat).

## Setup (this machine)

```sh
uv sync
cp config/settings.example.yaml config/settings.yaml
cp config/accounts.example.yaml config/accounts.yaml
cp config/rules.example.yaml config/rules.yaml
# edit the three yaml files
```

## Usage

```sh
# 1. Authorize each account (browser opens; one consent per account)
uv run gmailcleaner auth add you@gmail.com
uv run gmailcleaner auth list

# 2. Mirror metadata to local SQLite (first run is a full sync)
uv run gmailcleaner sync you@gmail.com --full
uv run gmailcleaner status

# 3. Build a plan (rules + optional LLM classification), review it
uv run gmailcleaner plan you@gmail.com --all
uv run gmailcleaner plan you@gmail.com --all --json /tmp/plan.json

# 4. Apply ONLY after reviewing the dry-run output
uv run gmailcleaner apply you@gmail.com --plan /tmp/plan.json --apply

# 5. New-mail check (cron-friendly; no LLM unless --classify)
uv run gmailcleaner watch --classify --apply-safe

# 6. Compose
uv run gmailcleaner draft you@gmail.com --to a@b.c --subject "Hi" --body "..."
uv run gmailcleaner send  you@gmail.com --draft-id <id> --yes
```

## Deployment (scheduled watch on an LXC)

The watch runs the same toolkit without OpenCode. Tokens live in
`data/tokens/`; authorize on a machine with a browser (loopback redirect),
then copy the repo + `data/` to the LXC. Refresh tokens are long-lived once
the consent app is in Production.

```sh
# on the LXC, every 15 min (adjust path):
*/15 * * * * cd /opt/gmail-cleaner && uv run gmailcleaner watch --classify --apply-safe >> data/watch.log 2>&1
```

Note: VPN up on hydralisk breaks host→LXC networking; run the cron on the LXC
itself, not via the host. The LLM endpoint must be reachable from the LXC.

## Layout

```
gmailcleaner/        package (cli, oauth, gmail, store, sync, rules, plan, classify, apply, watch, report)
config/              example yamls committed; real ones + client_secret.json gitignored
data/                tokens, sqlite mirrors, audit JSONL, reports (gitignored)
tests/               offline unit tests (rules, plan, store, classify parsing)
```

## Audit trail

Every apply (dry-run or real) writes `data/audit/<date>.jsonl`:
timestamp, account, action, target ids, reason (rule name or `llm:<category>`),
and `dry_run` flag. Trash is never auto-applied by `watch`.
