# Routines — Worked Example: Create a One-Off Reminder Routine

> Captured 2026-04-25 from a working session.
> The routine being created here is a **one-off** that fires once at a specific UTC timestamp, posts a reminder, and auto-disables.
> All identifiers (account UUID, trigger ID, environment ID, the v4 UUID inside `events`) are placeholders — replace them when adapting.
> The example repo is the canonical public placeholder `octocat/hello-world`. Substitute your own when you adapt this template.

---

## The use case

You shipped a staged rollout to production at 09:00 local time. The deploy ramps to 100% over six hours. You want Claude to nudge you once at 15:00 local to check error rates, latency, and rollout progress — and either give the green light to lock the rollout in or recommend a rollback.

That kind of *"remind me later about a state I'm waiting on"* task is a textbook fit for a one-off routine: it fires once, costs ~no tokens (small reminder output), and is exempt from the daily routine-run cap.

Other shapes that fit the same pattern:

- *"In 48 hours, check whether the feature flag is still healthy at 50% and either ramp it to 100% or open a rollback PR."*
- *"On Monday at 9am, summarize the PRs merged over the weekend and post a digest."*
- *"In 14 days, check if the experiment metrics have stabilized; if so, open a cleanup PR removing the variant."*

---

## Input — the `create` request

This is the body sent to `RemoteTrigger` with `action: "create"`. Equivalent HTTP call: `POST /v1/code/triggers` (the `RemoteTrigger` tool wraps it with auth).

```json
{
  "name": "Acme prod rollout — 6h post-deploy metrics check",
  "run_once_at": "2026-04-25T19:00:00Z",
  "enabled": true,
  "job_config": {
    "ccr": {
      "environment_id": "env_01EXAMPLE1234567890ABCDEF",
      "session_context": {
        "model": "claude-sonnet-4-6",
        "sources": [
          { "git_repository": { "url": "https://github.com/octocat/hello-world" } }
        ],
        "allowed_tools": ["Read", "Bash", "Grep", "Glob"]
      },
      "events": [
        {
          "data": {
            "uuid": "a4d8e2f1-9c3b-4f5e-8a2d-1b6c7d8e9f01",
            "session_id": "",
            "type": "user",
            "parent_tool_use_id": null,
            "message": {
              "role": "user",
              "content": "REMINDER — Acme prod rollout, 6 hours post-deploy.\n\nContext (snapshot at deploy time, 2026-04-25 09:00 UTC):\n- Release tag v3.7.0 was rolled out via the staged-deploy pipeline at 09:00 UTC.\n- Ramp schedule: 10% → 25% → 50% → 100%, advancing every 90 minutes if error rate < 1.0%.\n- Owner: deploys@acme.example.\n- Dashboards (the user has these bookmarked, you cannot reach them): Datadog 'acme-prod-overview' and Sentry project 'acme-api'.\n\nYour job (you cannot query Datadog, Sentry, or our prod metrics yourself; do NOT pretend to):\n\n1. Output a concise reminder to the user, written as a direct nudge — not a status report. Cover:\n   - It has been ~6 hours since v3.7.0 was deployed; ramp should be at or near 100%.\n   - They should check Datadog 'acme-prod-overview' for error rate + p95 latency on /api endpoints, and Sentry 'acme-api' for any new issue groups.\n   - Three possible states and what to do for each:\n     a) Metrics healthy → confirm the rollout, post 'rollout green' in the deploy channel.\n     b) Borderline (mild regression, < 2x baseline error rate) → leave at current ramp, schedule another check in 2 hours.\n     c) Regression (> 2x baseline error rate, p95 > SLO, or new prod-impacting Sentry group) → roll back via the deploy pipeline immediately.\n\n2. After they confirm the call (state a or c), suggest they reply with the outcome so the next session can:\n   - Update CHANGELOG.md with the post-rollout status note.\n   - Close out the rollout-tracking issue.\n   - Optionally draft a short retro entry for the team wiki.\n\n3. Optionally offer to draft the 'rollout green' deploy-channel message NOW (so it's ready to paste the moment metrics check out), highlighting v3.7.0 highlights from the release tag.\n\nKeep the message short and friendly. Do NOT spend tokens reading repo files unless the user asks you to draft the message. End with a single, clear call-to-action: 'Open Datadog acme-prod-overview now.'"
            }
          }
        }
      ]
    }
  }
}
```

### Top-level fields

| Field | Type | Required | What it means | Notes / gotchas |
|---|---|---|---|---|
| **`name`** | string | yes | Human-readable label shown in the Routines list at https://claude.ai/code/routines and in the CLI's `/schedule list`. | No length limit documented, but keep it scannable. Search-by-name in the web UI is exact-substring. |
| **`run_once_at`** | RFC3339 UTC string | one-of | Fires once at this timestamp, then auto-disables (`ended_reason: "run_once_fired"`). | **Must be in the future at create time.** The runtime rejects past timestamps. Mutually exclusive with `cron_expression`. |
| **`cron_expression`** | 5-field cron, UTC | one-of | Recurring schedule. Mutually exclusive with `run_once_at`. | **Min interval 1 hour** — `*/30 * * * *` is rejected. |
| **`enabled`** | boolean | optional, default `true` | Set to `false` to create a paused routine. | One-offs auto-disable after firing regardless. |
| **`mcp_connections`** | array | optional | MCP connectors attached to every run (Slack, Linear, Google Drive, etc.). | Empty here because the reminder doesn't need any. Names must match `[a-zA-Z0-9_-]` only — no dots or spaces. |
| **`job_config`** | object | yes | What the run actually does. Currently always wrapped in `{"ccr": {...}}` (Cloud Claude Runtime). | This is the only `job_config` shape supported by routines today. |

### Inside `job_config.ccr`

| Field | Type | What it means | Notes / gotchas |
|---|---|---|---|
| **`environment_id`** | `env_…` ID | The cloud environment (network rules, env vars, setup script) this run executes inside. | Every account gets a `Default` env auto-provisioned on first use. Custom envs are managed at https://claude.ai/code/environments. The session reads no local `~/`-files — env vars must be set on the environment, not your shell. |
| **`session_context`** | object | The session's "starting conditions" — model, repos, allowed tool list. | See sub-table below. |
| **`events`** | array | The list of pseudo-user-messages the session starts with. The first one's `message.content` is effectively the prompt. | Currently always 1 element. Each event must have a unique lowercase v4 UUID. |

### Inside `job_config.ccr.session_context`

| Field | Type | What it means | Notes / gotchas |
|---|---|---|---|
| **`model`** | model ID | Which Claude model to use for every run of this routine. | Default `claude-sonnet-4-6` for general work. Use `claude-opus-4-7` for harder reasoning, `claude-haiku-4-5-20251001` for cheap/fast. The selected model bills per its standard subscription cost. |
| **`sources`** | array of `{git_repository: {url}}` | Repos cloned at the start of every run. Each is checked out to its default branch. | Pushing requires the Claude GitHub App or `/web-setup` GitHub auth. By default Claude can only push to `claude/`-prefixed branches; toggle "Allow unrestricted branch pushes" per repo if needed. |
| **`allowed_tools`** | string[] | Whitelist of tools the cloud session may call. | Pruning this list shrinks the attack surface and the system prompt. The example only allows read-only tools (Read, Bash, Grep, Glob) since the reminder doesn't write files. Add `Edit`, `Write`, `WebSearch` if your routine needs them. |

### Inside `job_config.ccr.events[0].data`

| Field | Type | What it means | Notes / gotchas |
|---|---|---|---|
| **`uuid`** | lowercase v4 UUID string | Unique ID for this seed-message. | You generate it. Reusing UUIDs across creates is undefined behavior — generate a fresh one each time. |
| **`session_id`** | string | Empty `""` at create time. Filled by the runtime when a run actually starts. | Don't pre-fill. |
| **`type`** | `"user"` | Marks this seed-message as if it came from the user. | Always `"user"` for the kickoff event. |
| **`parent_tool_use_id`** | string \| null | Threads this event under a tool-use turn. | Always `null` for the kickoff event. |
| **`message.role`** | `"user"` | Same role-shape as a regular Claude API message. | Mirrors `type` above. |
| **`message.content`** | string | **The prompt.** This is what the cloud session "wakes up to." | The single most important field — see "Prompt design" below. |

---

## Prompt design — what makes the example prompt work

The prompt above packs five pieces into one self-contained brief:

1. **Snapshot context.** Date/time of the deploy, the release tag, the ramp schedule, the relevant dashboards. The cloud session has zero conversation memory — without this, it has no anchor for "what state was true when the routine was scheduled."
2. **Hard constraints on what the agent can/can't do.** *"You cannot query Datadog, Sentry, or our prod metrics yourself; do NOT pretend to"* heads off hallucinated status reports — a common failure mode for reminder agents that try to act like dashboards.
3. **A branched response.** Three concrete states (Healthy / Borderline / Regression) with the action for each. No ambiguity about what "good" output looks like, and no need for the agent to invent a decision framework.
4. **A pre-loaded follow-up.** *"After they confirm the call, suggest they reply with the outcome so the next session can…"* — primes the user to give the next conversation enough signal to update the changelog and close the rollout-tracking issue without re-establishing context.
5. **A single clear CTA.** *"End with: Open Datadog acme-prod-overview now."* — bounds the output so the agent doesn't ramble.

When you write your own routine prompts, treat them as **one-shot briefs to a colleague who just walked into the room**. Anything implicit in your current chat is invisible to the cloud session.

---

## Output — the `create` response

A successful `RemoteTrigger.create` returns HTTP 200 with the saved routine echoed back:

```json
{
  "trigger": {
    "id": "trig_01ABCDEFGHJKLMNOPQRSTUVW",
    "name": "Acme prod rollout — 6h post-deploy metrics check",
    "enabled": true,
    "cron_expression": "",
    "run_once_at": "2026-04-25T19:00:00Z",
    "next_run_at": "2026-04-25T19:00:00Z",
    "ended_reason": "",
    "persist_session": false,
    "api_token_hint": "",
    "enabled_plugins": [],
    "extra_marketplaces": [],
    "mcp_connections": [],
    "creator": {
      "account_uuid": "<your_account_uuid>",
      "display_name": "<your_display_name>"
    },
    "created_at": "2026-04-25T11:33:04.735874Z",
    "updated_at": "2026-04-25T11:33:04.735874Z",
    "job_config": { "ccr": { "...echoed back from input": "..." } }
  }
}
```

### Response fields

| Field | What it means | Why you might care |
|---|---|---|
| **`trigger.id`** | Opaque routine ID, prefix `trig_`. | The handle for every other operation: `RemoteTrigger.get`, `.update`, `.run`. Also the path segment in `https://claude.ai/code/routines/{id}` and in the API `/fire` endpoint. Save it. |
| **`name`** | Echoed from input. | — |
| **`enabled`** | Echoed from input. | — |
| **`cron_expression`** / **`run_once_at`** | Whichever you sent. The other is `""` / null. | — |
| **`next_run_at`** | UTC RFC3339 timestamp of the next firing. | For one-offs, equals `run_once_at`. For cron, the runtime computes the next match. After a one-off fires this becomes empty and `ended_reason` flips to `"run_once_fired"`. |
| **`ended_reason`** | `""` while alive. After a one-off fires: `"run_once_fired"`. After explicit disable: another reason string. | The web UI maps `"run_once_fired"` → "Ran" badge. |
| **`persist_session`** | `false` for routines today. | Each run gets a fresh session — no carry-over. Don't try to thread state through this field. |
| **`api_token_hint`** | Last few chars of the API trigger's bearer token, if one exists. Empty here because no API trigger was added. | If you later add an API trigger, the token is shown **once** at generation time — `api_token_hint` is the only thing you can compare against later to confirm which token is active. |
| **`enabled_plugins`** | Reserved / empty. | — |
| **`extra_marketplaces`** | Reserved / empty. | — |
| **`mcp_connections`** | Echoed from input. | — |
| **`creator.account_uuid`** | The claude.ai account that owns this routine. | Routines are per-account, not per-org. They don't share with teammates. |
| **`creator.display_name`** | Your claude.ai display name. | Anything the routine does via your GitHub identity or connectors appears as you. |
| **`created_at`** / **`updated_at`** | Microsecond-precision UTC timestamps. | `updated_at` advances on every `.update` call. |
| **`job_config`** | Echoed verbatim from input. | Useful for sanity-checking what the server actually stored, especially if you mass-create routines from a script. |

---

## Adapt this template — checklist

When reusing this shape for your own routine:

1. **Recompute `run_once_at`.** Always re-fetch `date -u` first; don't compose from conversation memory of "now." Confirm the local-time-→-UTC conversion with the user before sending.
2. **Pick the right `environment_id`.** If you need a private API key in the run, set it on the environment in the web UI first — don't paste it into the prompt.
3. **Trim `allowed_tools` to the minimum.** Read-only? `["Read", "Bash", "Grep", "Glob"]`. Edits? Add `Edit`. Web research? Add `WebSearch` / `WebFetch`. Cuts both context and risk.
4. **Generate a fresh `uuid`.** Lowercase v4. Don't reuse from a prior create.
5. **Front-load context in the prompt.** Date, version numbers, branch, commit SHAs, what state was true *at scheduling time*. The cloud session has no other source for any of it.
6. **Be explicit about what NOT to do.** "Don't query X you can't access," "don't read repo files unless the user asks." Saves tokens and prevents hallucinated reports.
7. **End with a single CTA.** A bounded response is a useful response.
8. **For one-offs: don't push `cron_expression`. For cron: don't push `run_once_at`.** They're mutually exclusive — sending both is a 400.
9. **Save the `trigger.id`** that comes back. You can also find it later at https://claude.ai/code/routines, but capturing it from the create response is the cheapest path.

---

## Related

- [Routines — Billing & Limits](billing-and-limits.md) — what the runs cost and the daily caps
- Official docs: https://code.claude.com/docs/en/web-scheduled-tasks
- API ref for the `/fire` endpoint (API triggers): https://platform.claude.com/docs/en/api/claude-code/routines-fire
