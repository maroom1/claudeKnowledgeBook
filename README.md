# Claude Knowledge Book

Personal reference notes about Claude Code features, behaviors, and limits — captured from working sessions and verified against official Anthropic documentation.

This is a knowledge base, not project code. Each document is self-contained and dated at the top so I can tell when the information might be stale.

## Topics

- [Routines — Billing & Limits](routines/billing-and-limits.md) — How scheduled remote agents are billed, daily run caps per plan, one-off run rules, GitHub/API triggers.
- [Routines — Worked Example: Create a One-Off Reminder Routine](routines/example-create-one-off.md) — Full `RemoteTrigger.create` request and response, with every field annotated.
- [Routines — Authentication](routines/authentication.md) — When you need to pass a bearer token (and when you don't): `RemoteTrigger`, scheduled / cron / one-off, GitHub trigger, external `/fire`.

## Conventions

- Each note carries a "Captured `YYYY-MM-DD`" line at the top with source links. Reverify before relying on details older than a few months — Claude Code changes fast.
- Topic-level folders (`routines/`, `costs/`, `mcp/`, etc.) keep things scannable as the book grows.
- Plain markdown, no front matter, no build step. GitHub renders it directly.
