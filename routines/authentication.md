# Routines — Authentication

> Captured 2026-04-25.
> Verified against:
> - https://code.claude.com/docs/en/web-scheduled-tasks
> - https://platform.claude.com/docs/en/api/claude-code/routines-fire

There are several distinct paths that "call" or "fire" a routine, and each has a different auth model. The single most common confusion is conflating the in-process `RemoteTrigger` tool calls (no token from you) with the external `/fire` HTTP endpoint (per-routine bearer token).

---

## TL;DR — when do I need to pass a token?

| Scenario | Token from you? | What's authenticating |
|---|---|---|
| **`RemoteTrigger` tool calls** (`create`, `get`, `list`, `update`, `run`) from inside a Claude Code session | ❌ No | Your claude.ai OAuth session — the tool injects it in-process |
| **Scheduled run firing** (`cron_expression` ticks, `run_once_at` timestamp arrives) | ❌ No | Internal Anthropic-cloud trigger, no external auth involved |
| **GitHub-triggered run firing** (PR/release event matches) | ❌ No | The Claude GitHub App's webhook delivery is what authenticates the event; you must install the app on the repo first |
| **External `POST /fire`** from your own scripts, alerting tools, CI, internal dashboards | ✅ **Yes** | Per-routine bearer token, generated once in the web UI |
| **Cloning the source repo** at run start | ❌ Indirect | The routine's owner-account identity, established via `/web-setup` or the Claude GitHub App at install time |
| **Connector calls** (Slack, Linear, Google Drive, etc.) during a run | ❌ Indirect | The routine's owner-account's connector OAuth tokens — actions appear as you |

---

## 1. `RemoteTrigger` tool calls — no token from you

The `RemoteTrigger` tool exposed inside Claude Code wraps the routines API (`POST /v1/code/triggers`, `GET /v1/code/triggers/{id}`, etc.). Its description says:

> Use this instead of curl — the OAuth token is added automatically in-process and never exposed.

So when Claude calls `RemoteTrigger.create`, the runtime resolves your existing claude.ai OAuth session and forwards it as the auth header server-side. You don't generate, paste, or even see a token.

Implications:

- **Don't curl `/v1/code/triggers/...` directly to manage routines.** The OAuth token isn't generally available outside the tool's process. Use `RemoteTrigger` if you're inside a Claude Code session, or use the web UI at https://claude.ai/code/routines.
- **Don't try to forward this OAuth session into an external script.** That session is short-lived and tied to your interactive login. The right tool for "external script wants to fire a routine" is the per-routine bearer token in path 4 below, not your OAuth session.

---

## 2. Scheduled / cron / one-off runs — no token

When `run_once_at` arrives or a `cron_expression` matches, the Anthropic-cloud scheduler fires the routine internally. There's no incoming HTTP request from you, so there's nothing to authenticate at the outer layer.

The run *itself* still authenticates outward:

- The clone of `sources[].git_repository.url` uses your linked GitHub identity (set up via `/web-setup` or the Claude GitHub App).
- Each connector call uses your linked OAuth for that connector.
- Any commit / PR / Slack message therefore appears as **you**.

---

## 3. GitHub event triggers — no token, but install required

When a `pull_request.opened` (etc.) matches a routine's GitHub trigger, GitHub delivers the event via webhook to Anthropic's cloud. The auth here is the GitHub App's signed delivery — not anything you handle.

What *you* must do beforehand:

- **Install the Claude GitHub App** on the repository. Without it, no webhooks reach Anthropic, and the trigger never fires.
- `/web-setup` in the CLI grants repo-clone access but **does not** install the GitHub App and **does not** enable webhook delivery. The trigger setup flow on the web prompts you to install.

Quotas during the research preview: per-routine and per-account hourly caps. Events beyond the limit are dropped until the window resets.

---

## 4. External `POST /fire` — per-routine bearer token (the only path where *you* hand over a token)

This is the path you'd use to fire a routine from CI, an alerting webhook, an internal dashboard, or any non-Claude HTTP client.

### Generating the token

1. Go to the routine at `https://claude.ai/code/routines/<trigger_id>`.
2. Click the pencil icon → **Edit routine** → scroll to **Select a trigger** → **Add another trigger** → **API**.
3. The modal shows:
   - The full endpoint URL (`https://api.anthropic.com/v1/claude_code/routines/<trigger_id>/fire`)
   - A sample `curl` command
   - A **Generate token** button — click it.
4. **The token is shown ONCE.** Copy it immediately to your secret store (alerting tool, CI secret, password manager). It cannot be retrieved later. Only the last few characters of the token are recoverable, surfaced as `api_token_hint` in the routine's JSON, so you can confirm which token a stored secret matches.

The CLI cannot generate or revoke API trigger tokens — only the web UI does.

### Calling the endpoint

```bash
curl -X POST https://api.anthropic.com/v1/claude_code/routines/<trigger_id>/fire \
  -H "Authorization: Bearer sk-ant-oat01-xxxxx" \
  -H "anthropic-beta: experimental-cc-routine-2026-04-01" \
  -H "anthropic-version: 2023-06-01" \
  -H "Content-Type: application/json" \
  -d '{"text": "Sentry alert SEN-4521 fired in prod. Stack trace attached."}'
```

Required headers:

| Header | Value | Purpose |
|---|---|---|
| `Authorization` | `Bearer <token>` | Per-routine bearer token from step 4 above |
| `anthropic-beta` | `experimental-cc-routine-2026-04-01` | Opts in to the research-preview surface |
| `anthropic-version` | `2023-06-01` | Standard Anthropic API version pin |
| `Content-Type` | `application/json` | Standard |

The body's `text` field is **freeform string** — passed alongside the routine's saved prompt. If you POST JSON, the routine receives it as a literal string, not parsed.

### Successful response

```json
{
  "type": "routine_fire",
  "claude_code_session_id": "session_01HJKLMNOPQRSTUVWXYZ",
  "claude_code_session_url": "https://claude.ai/code/session_01HJKLMNOPQRSTUVWXYZ"
}
```

Open the session URL in a browser to watch the run live, review the changes, or continue the conversation manually.

### Token lifecycle

- **Per-routine, single-purpose.** Each token can only fire its own routine. Stealing one token from one routine doesn't grant any access to another.
- **Rotate** — return to the same modal, click **Regenerate**. The old token is invalidated immediately; update your secret store.
- **Revoke** — same modal, click **Revoke**. The routine survives, but the API trigger is removed and any external scripts using the old token start getting 401s.
- **No cross-routine secret reuse.** Don't try to share one token across multiple routines — Anthropic doesn't offer that, and you'd lose the per-routine revoke story.

### Beta-header versioning

The `/fire` endpoint ships under the `experimental-cc-routine-2026-04-01` beta header. Request/response shapes, rate limits, and token semantics may change while in research preview. Anthropic's policy: breaking changes ship behind new dated beta-header versions, and the **two most recent prior versions** continue to work so callers have time to migrate. Watch the dated header value when bumping integrations.

### Where the `/fire` endpoint lives

The `/fire` endpoint is part of the **claude.ai user surface** (`api.anthropic.com/v1/claude_code/...`), not the standard Claude Platform API surface. If you're already using the Claude Platform API for inference, that's a different auth context entirely — don't reuse those `sk-ant-api…` keys here.

---

## What the routine itself authenticates outward (during a run)

Even when no token is exchanged at the trigger boundary, the running session still acts as you toward outside services:

| Action during a run | Authenticated as |
|---|---|
| Cloning a `sources[]` GitHub repo | Your linked GitHub identity (via `/web-setup` or the Claude GitHub App install) |
| Pushing to a `claude/`-prefixed branch | Same — commits show your GitHub username |
| Pushing to a non-`claude/` branch | Only allowed if **Allow unrestricted branch pushes** is enabled on the repo for this routine |
| Slack message via the Slack connector | Your linked Slack OAuth |
| Linear ticket via the Linear connector | Your linked Linear OAuth |
| Google Drive read/write via the Drive connector | Your linked Google OAuth |
| Generic `WebFetch` / `WebSearch` | Anthropic's egress, no per-user auth |
| Calling the Claude Platform API from inside the run | Whatever `ANTHROPIC_API_KEY` is set on the **environment**, not your laptop |

So: **routines don't need a token from you to fire** (except path 4), but they **do act as you** toward every external service they touch. Scope each connector and each repo's branch-push setting accordingly.

---

## Practical recipes

### "I want a CI job to fire a routine after every successful deploy"

→ Path 4. Generate an API trigger token, store it in your CI secret manager (`ROUTINE_FIRE_TOKEN`), then in the deploy job: `curl -X POST ...$ROUTINE_FIRE_TOKEN`. Use the `text` body to pass the deploy SHA / version.

### "I want my Sentry alert webhook to spawn a triage routine"

→ Path 4. Same pattern — Sentry webhook → `POST /fire` with the alert body in `text`.

### "I want a routine to run nightly without me involved"

→ Path 2 (cron). No token. Set `cron_expression: "0 9 * * *"` (UTC) and walk away.

### "I want a routine to run when a PR is opened"

→ Path 3 (GitHub trigger). No token, but install the Claude GitHub App on the repo first.

### "I want my locally-running script to programmatically create new routines"

→ Path 1 only — and only if the script is invoking Claude Code, since that's what makes `RemoteTrigger` available. There is no documented public API for creating routines from outside Claude Code today; if you have a multi-routine fleet to manage, the supported path is the web UI.

---

## Related

- [Routines — Billing & Limits](billing-and-limits.md) — what runs cost and the daily caps
- [Routines — Worked Example: Create a One-Off Reminder Routine](example-create-one-off.md) — full annotated `create` request and response
- Official docs: https://code.claude.com/docs/en/web-scheduled-tasks
- API ref for the `/fire` endpoint: https://platform.claude.com/docs/en/api/claude-code/routines-fire
