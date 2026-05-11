# Lesson — Don't /rrr until inbox is silent

**Date**: 2026-05-10
**Source session**: `a7679a83` (continuation) — supplemental retro after premature wrap
**Domain**: session hygiene · multi-agent communication

## Pattern

A retrospective is a **snapshot of completed work**, not a **curtain call** that ends the session. Trigger /rrr when external signals say "done", not when internal feelings say "I'm tired" or "this seems like a stopping point."

**Better trigger conditions** (all should hold):
- Inbox is silent (no unread peer-to-peer messages)
- No threads where I'm blocked-on-me (no GitHub issue / PR / DM awaiting my reply)
- Explicit close signal from the human or coordinator agent ("ราตรีสวัสดิ์", "wrap up", "done for today")
- No active task with a deadline within the next hour

**If work continues after /rrr**: write a supplemental retro for the new slice. Don't pretend it didn't happen. Don't fold it into the previous retro silently.

## Why it works

- **Honest reflection requires complete data** — a retro written before the day's last task is structurally incomplete. The lessons it draws are partial.
- **External signals are more reliable than internal "feels done"** — internal pressure to wrap rises with hours-on-task; external task arrival is independent.
- **Supplemental retros preserve the timeline** — if 40 more minutes of work happened, you want that captured separately so future-you can see the actual session shape, not a smoothed version.
- **Avoids self-fulfilling prophecy** — declaring "wrap" before work stops can make you reluctant to engage incoming work ("but I already wrapped"). Letting the inbox dictate the wrap removes that friction.

## When the rule bends

- **Hard deadline**: if the human says "wrap by 23:00", do /rrr at 22:55 even if inbox isn't silent — flag the inbox state in Honest Feedback and write a supplemental later.
- **Long sessions where checkpointing matters**: do periodic `/rrr` checkpoints every 2–3 hours of active work. Each is honest about being a checkpoint, not a wrap. Final wrap retro consolidates.
- **Background work**: if you're waiting on long-running async tasks, /rrr the synchronous portion now — clearly title it as such ("partial: pre-build wrap").

## Anti-pattern to avoid

Writing a retro that says "Next Steps: [tomorrow's plan]" then doing those steps in the same session. Either the retro is wrong, or the "next session" framing is. Today: 22:44 retro listed "Tomorrow morning: pseudocode pass-2" — but engineering review + commits + PR happened *this* session, not next. The Next Steps section was honest at write-time but became misleading 13 minutes later.

## Evidence

This session: /rrr at 22:44 said "ราตรีสวัสดิ์". Then John's coupling check arrived at 22:57 → engineering response. Then fluke requested commit at 23:10 → branch + 3 commits + push + [PR #3](https://github.com/flukeworachai/forge-oracle/pull/3). Two more maw rounds with John. The "wrap" was a misread by 40 minutes and ~200+ lines of substantive output.

Supplemental retro at 23:24 captures it honestly. Combined retros = real session shape.
