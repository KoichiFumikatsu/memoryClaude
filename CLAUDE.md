# Claude Memory System

## Session Initialization

At the start of every session, read the following files before doing anything else:

- [memory/personality.md](memory/personality.md) — **read first** — defines how to think, communicate, and make decisions for this user
- [memory/user.md](memory/user.md) — who the user is, their role, expertise, and goals
- [memory/preferences.md](memory/preferences.md) — how the user wants Claude to behave
- [memory/decisions.md](memory/decisions.md) — past decisions so they don't get re-litigated
- [memory/people.md](memory/people.md) — collaborators and contacts mentioned across sessions

## Session End

Before finishing any session, update the relevant memory files with:
- New facts learned about the user → `memory/user.md`
- New preferences or corrections given → `memory/preferences.md`
- Important decisions taken → `memory/decisions.md`
- New people mentioned → `memory/people.md`

Then the stop hook will auto-commit and push any changes.

## Memory Update Rules

- Only write what is **non-obvious** and **reusable** across sessions.
- Do not store ephemeral task state (use TodoWrite for that).
- Do not store what is already in git history or the code itself.
- Always convert relative dates to absolute dates (e.g. "Thursday" → "2026-05-08").
- When a memory conflicts with current reality, update the file — don't act on stale data.
