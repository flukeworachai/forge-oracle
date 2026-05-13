# CODESYS Chiller Plant Spike

> Forge spike workspace for [atlas-oracle#3](https://github.com/flukeworachai/atlas-oracle/issues/3).
> Source design: `atlas-oracle/ψ/learn/factory-chiller-plant/` (read-only — Atlas authored).

## Scope

3-gate spike — local-only, no hardware:

1. **Gate 1** — CODESYS Control Win runtime + 25 tags in GVL + WebVisu live at `localhost:8080/webvisu.htm`
2. **Gate 2** — Postgres + TimescaleDB up + ingest service writes to `tag_history`
3. **Gate 3** — Grafana dashboard + ML IEC Library kW anomaly detection

Decision note (closing): ESP32 + Node-RED + DB + Power BI **vs** CODESYS Control Win + WebVisu + Timescale + Grafana — when does each win.

## Layout

| What | Where |
|------|-------|
| CODESYS `.project` file | `C:\Users\flukw\Documents\CODESYS Projects\chiller-plant-spike\` (Windows side, untouched) |
| Spike notes + scripts | this dir (`forge-oracle/ψ/lab/codesys-chiller-spike/`) |
| Source of truth (tags, schema, README) | `atlas-oracle/ψ/learn/factory-chiller-plant/` (read-only) |
| WebVisu | `http://localhost:8080/webvisu.htm` |

## Notes index

- `setup-friction.md` — running log of install pain points (decision-note input)
- `gate1-instructions.md` — step-by-step batches sent to fluke (also mirrored on issue #3)

## Pacing

Atlas confirmed option (b) — guide live. Forge sends 3-5 step batches → fluke executes → pastes output / screenshot → Forge verifies via `/mnt/c/` + sends next batch. Friction observed first-hand = decision note ลึก.
