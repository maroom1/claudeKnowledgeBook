# Claude Code Routines — Billing & Limits

> Captured 2026-04-25 from a working session. Verified against:
> - https://code.claude.com/docs/en/web-scheduled-tasks
> - https://code.claude.com/docs/en/costs
> - https://claude.com/blog/introducing-routines-in-claude-code

## What is a routine?

A routine is a saved Claude Code configuration — a prompt, one or more repositories, a set of MCP connectors, and one or more triggers — packaged once and run automatically on Anthropic-managed cloud infrastructure. Routines keep working when your laptop is closed.

A single routine can have **multiple triggers** combined:

| Trigger | Fires when |
|---|---|
| **Scheduled** | Recurring cadence (cron, min interval 1 hour) OR a single one-off `run_once_at` timestamp |
| **API** | An authenticated POST hits the routine's `/fire` endpoint with a bearer token |
| **GitHub** | A repo event matches (`pull_request.opened`, `release.published`, etc., with optional filters) |

Each run spawns an isolated cloud session (CCR) with its own git checkout. State is **not** carried between runs — every prompt must be self-contained.

## Billing — TL;DR

> **Routines draw down subscription usage the same way interactive sessions do.** *(Source: Claude Code docs)*

So if you're on **Pro / Max / Team / Enterprise**, a routine run consumes the same subscription quota as a normal Claude Code session. It does **not** bill via separate API credits.

If you're on direct API billing (no subscription), routine runs bill via your API key like any other call.

## Daily routine-run caps (separate from token usage)

| Plan | Routine runs per day |
|---|---|
| Pro | **5** |
| Max | **15** |
| Team / Enterprise | **25** |

These caps are **on top of** the regular subscription token limits. Hitting either limit blocks further runs until the window resets, **unless** the org has **Extra Usage** enabled — then runs continue on metered overage.

Enable Extra Usage at: **Settings → Billing** on claude.ai.

## One-off runs (`run_once_at`) are special

> One-off runs do **not** count against the daily routine run cap. They draw down regular subscription usage like any other session, but are **exempt** from the per-account daily routine allowance.

So a `/schedule in 6 hours, ping me to check X` style reminder is "free" against your daily-routine-cap budget. The token cost still applies.

After it fires, the routine **auto-disables** and the web UI marks it as **Ran**. To run it again, edit the routine and set a new `run_once_at`.

## What can / can't a routine reach?

A routine starts with **zero local context** — no laptop, no env vars, no `~/`-based files. What it can reach:

| Resource | How it gets there |
|---|---|
| GitHub repo files | Cloned from the URL(s) listed in `sources` at run start |
| MCP connectors (Slack, Linear, Google Drive, etc.) | Listed in `mcp_connections` on the routine |
| Network | Determined by the **environment** (env_id) |
| Env vars / secrets | Set on the environment — **not** copied from your local shell |
| Skills | Only those committed to the cloned repo |

Push permission: by default Claude can push only to `claude/`-prefixed branches. To allow other branches, enable **Allow unrestricted branch pushes** per repo on the routine.

## Identity — actions appear as you

Anything a routine does through your connected GitHub identity or connectors **appears as you**. Commits and PRs carry your GitHub user; Slack messages, Linear tickets, etc. use your linked accounts. Routines are not shared with teammates, and they count against *your* daily run allowance.

## Schedule via CLI

`/schedule` in any session creates **scheduled** routines (recurring or one-off). It cannot create API or GitHub triggers — those are added on the web at https://claude.ai/code/routines.

Examples (from the docs):

```text
/schedule daily PR review at 9am
/schedule in 2 weeks, open a cleanup PR that removes the feature flag
/schedule tomorrow at 9am, summarize yesterday's merged PRs
```

Cron expressions are stored in **UTC** but the CLI accepts local time and converts (showing both, asking confirmation). Minimum interval: **1 hour**. `*/30 * * * *` is rejected.

CLI management: `/schedule list`, `/schedule update`, `/schedule run`. There is **no `/schedule delete`** — delete from https://claude.ai/code/routines.

## API trigger — calling `/fire`

Once an API trigger is added (web UI only), POST to:

```bash
curl -X POST https://api.anthropic.com/v1/claude_code/routines/{trigger_id}/fire \
  -H "Authorization: Bearer sk-ant-oat01-xxxxx" \
  -H "anthropic-beta: experimental-cc-routine-2026-04-01" \
  -H "anthropic-version: 2023-06-01" \
  -H "Content-Type: application/json" \
  -d '{"text": "freeform context — passed to the routine alongside its saved prompt"}'
```

The `text` field is **freeform string** — JSON sent here is received as a literal string, not parsed.

Response:

```json
{
  "type": "routine_fire",
  "claude_code_session_id": "session_01HJK...",
  "claude_code_session_url": "https://claude.ai/code/session_01HJK..."
}
```

The `/fire` endpoint is in research preview behind the `experimental-cc-routine-2026-04-01` beta header. Two prior beta-header versions remain valid during deprecation windows.

## GitHub triggers — caveats

- Requires the **Claude GitHub App** installed on the repo (not just `/web-setup` clone access).
- Hourly per-routine and per-account caps during research preview — events beyond the cap are dropped until the window resets.
- Each matching event starts its **own** session — no session reuse across events.
- Supported event categories: `pull_request.*`, `release.*`. Filters: author, title, body, base/head branch, labels, is-draft, is-merged. Operators: equals, contains, starts with, is one of, is not one of, regex.
- `matches regex` tests the **entire** field — wrap with `.*` for substring matching, or use `contains`.

## Watching usage

- https://claude.ai/code/routines — daily routine-run counter
- https://claude.ai/settings/usage — overall subscription usage
- `/usage` in any CLI session — per-session token counter (subscription users see plan bars; the dollar figure is API-only)

## Quick checklist when scheduling

1. Will the agent need a private repo? → Run `/web-setup` first.
2. Will it open PRs / push? → Confirm GitHub connected; consider unrestricted-branch-push only if needed.
3. Will it touch Slack / Linear / Google Drive? → Connect those MCP connectors at https://claude.ai/customize/connectors before creating the routine.
4. Does the prompt assume any local state (env vars, paths, prior conversation)? → Rewrite — the cloud session has none of it.
5. One-off or recurring? → One-off doesn't burn the daily cap.
6. Is the work genuinely worth the token spend each run? → A 5/day cap on Pro fills up fast.
