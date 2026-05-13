# Lesson — Enumerate design decisions for review

**Date**: 2026-05-10
**Source session**: `a7679a83` — CODESYS Chiller Spike prep day
**Domain**: cross-Oracle collaboration · review protocols

## Pattern

When delivering work to a reviewer (human or another Oracle), present design decisions as a **numbered list with explicit reasoning per item**, not buried in prose.

```
### Design decisions (flag before changing)

1. **<Decision>** — <one-line reason> + <alternative considered, why rejected>
2. **<Decision>** — ...
```

The reviewer can endorse / reject / request-clarify each item by number. Round-trips collapse from "I have concerns about your approach" to "5/5 endorse + sync README §2."

## Why it works

- **Coverage** — reviewer can't accidentally skip a decision; numbering forces eye-pass over every item
- **Targeted disagreement** — "object to #2 only" vs vague "I have concerns"
- **Audit trail** — endorsed list becomes the contract for the next phase
- **Async-friendly** — works whether reviewer responds in 5 minutes or 5 days

## When to use

- Routed tasks where another agent / human must approve before you proceed
- Spike prep where decisions baked in early are expensive to undo later
- Any deliverable where the reviewer is time-constrained (numbering = lower review cost = faster turnaround)

## When NOT to use

- One-off bug fixes — overhead > value
- Decisions with no alternative (forced by constraint) — frame as "constraints surfaced" instead
- When the work itself is the artifact under review (e.g., a code diff — the diff is the list)

## Evidence

This session: 5 design decisions flagged in [issue #3 chip-in](https://github.com/flukeworachai/atlas-oracle/issues/3#issuecomment-4415654959) → Atlas review at [comment 4415670980](https://github.com/flukeworachai/atlas-oracle/issues/3#issuecomment-4415670980) endorsed all 5 + identified one upstream sync action, all in one round-trip.
