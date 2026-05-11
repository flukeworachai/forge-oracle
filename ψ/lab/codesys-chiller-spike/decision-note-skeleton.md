# Decision Note — CODESYS stack vs ESP32 stack (skeleton)

> Closing artifact for [atlas-oracle#3](https://github.com/flukeworachai/atlas-oracle/issues/3). Filled progressively from `setup-friction.md` + observations during gates 1–3. Posted as final comment on issue #3 once gate 3 passes.

---

## TL;DR (one paragraph, fill last)

> _< 5 lines: which stack wins for which class of project, with the dominant factor named. Example skeleton:_
> _"ESP32 stack wins for **<class A>** because **<dominant factor>**. CODESYS stack wins for **<class B>** because **<dominant factor>**. Crossover point is at **<axis>**."_

## Compared stacks

| Layer | Stack A — "ESP32 path" | Stack B — "CODESYS path" |
|-------|-----------------------|--------------------------|
| Sensor / edge | ESP32 + custom firmware | CODESYS Control Win (PLC runtime) |
| Logic | C / MicroPython | IEC 61131-3 (ST/LD/FBD) |
| Control loop determinism | best-effort (RTOS) | scan-cycle deterministic |
| HMI / dashboard | Node-RED dashboard | WebVisu HTML5 (built-in) |
| Historian | DB write from Node-RED | OPC UA → ingest service → Postgres+Timescale |
| Analytics dashboard | Power BI (cloud) | Grafana (Timescale-native) |
| Anomaly detection | Python notebook / cloud ML | ML IEC Library in scan cycle |
| Cost (license) | $0 | Free Dev System; Control Win SL paid prod, trial dev |
| Vendor lock | mostly open | IEC 61131-3 portable across CODESYS-flavored vendors (WAGO/Beckhoff/Schneider/ABB) |

## Evaluation matrix

### 1. Setup friction

> _Fill from `setup-friction.md`. Quantify: minutes from "no software" → "first variable on screen". Note any blockers that needed escalation (license email wait, port conflict, AV, missing redistributable, etc.)._

| Step | ESP32 path | CODESYS path |
|------|-----------|--------------|
| Account / license | _from observation_ | _from observation_ |
| Install (size, time) | _N/A or recall_ | _from observation_ |
| First-run gotchas | _N/A or recall_ | _from observation_ |
| First "hello world" tag visible | _N/A or recall_ | _from observation_ |
| **Total to gate 1** | _est._ | _measured_ |

### 2. WebVisu DX vs Node-RED dashboard

> _Filled during gate 2. Compare on five axes:_

- **Drag-drop fidelity** — how close to "what you draw is what you get"
- **Bind-to-tag effort** — clicks per tag wired
- **Animation / dynamic visual** — gauges, color states, alarm flash
- **Page navigation / multi-screen** — mimic + alarm + trend in one app
- **Export / share** — URL only, file export, embedding into another app

| Axis | Node-RED | WebVisu |
|------|----------|---------|
| ... | ... | ... |

### 3. ML library realism

> _Filled during gate 3. The ML IEC Library claim is "anomaly detection in PLC scan cycle". Assess:_

- Use-case fit (drift / spike / pattern) — which detected, which missed?
- Training workflow (in-IDE? export-import? size of dataset to baseline?)
- Inference cost (scan-cycle impact at 1 s tick?)
- Custom-model story (can we drop in our own ONNX, or only library-provided algos?)

| Use-case | Worked out-of-box? | Effort to make it work | Notes |
|----------|-------------------|----------------------|-------|
| kW drift (gradual) | _from observation_ | _from observation_ | _from observation_ |
| Spike anomaly (one-tick) | _from observation_ | _from observation_ | _from observation_ |
| Pattern (multi-variable) | _from observation_ | _from observation_ | _from observation_ |

### 4. Deployment story (vendor-portability + production)

> _What does it take to move from "running on fluke's laptop" to "running at a real site on real hardware"?_

- **CODESYS path**: same project compiled to WAGO PFC / Beckhoff CX / Schneider M2x / Phoenix EPC… does it actually work? Any I/O config that breaks portability? OPC UA endpoint exposable to off-site historian?
- **ESP32 path**: production deployment looks like flashing firmware to N devices, OTA strategy, fleet provisioning. Different problem class.
- **What surprised us** (1–2 lines from spike experience).

#### 4a. Pre-populated cross-trade discoveries (from John's coupling check, atlas-oracle#6)

These items were surfaced **before** the spike ran live — by John's `mep-coordinator` subagent reviewing the spike SUPPLY-side design against a real LOAD-side (mock 2-storey office, 25.5 TR). Filled-in observations from the spike will refine or contradict these.

| Discovery | What it means for deployment | Forge stance |
|-----------|-----------------------------|--------------|
| **Spike pump (475 GPM @ 150 kPa, const-speed) is wildly oversized for John's 68 GPM circuit** — at 25% turn-down a const-speed centrifugal sits hard left on its curve, head balloons 125–140% causing valve throttling + noise | For production: VFD primary control with DP sensor at remote AHU is **mandatory**, not a "later upgrade." Spike retains const-speed for ML signal simplicity but the production answer is different. | Pseudocode `FB_Pump` annotated with spike-vs-real comment (pass-2). Real-coupled sizing: ~85 GPM @ 90 kPa, ~2 kW VFD. |
| **ASHRAE 90.1-2022 §6.5.4.2** mandates variable-flow on primary CHW pumps serving > 3 modulating valves | Building energy code compliance is not optional. Spike const-speed would fail code review. | Document standard cite + payback math (1–1.5 yr at typical load) in decision note final. |
| **BAS protocol mismatch**: spike speaks OPC UA only; real BAS speaks BACnet/MSTP or BACnet/IP | Production runtime needs dual protocol exposure. **CODESYS V3 native `CmpBACnet` library** — same Control Win runtime serves both OPC UA + BACnet/IP simultaneously, no external gateway. If BAS is MSTP-only, ~$300 BACnet/IP-to-MSTP router (Contemporary Controls BASrouter) — Forge-owned. | Strong CODESYS-stack advantage: ESP32+Node-RED would require additional gateway tier (or Node-RED BACnet contrib) — extra moving part. |
| **Plant sized for spike scenario, not building load** — 100 TR plant for 25.5 TR building = chiller IPLV at 25–35% PLR, not rated COP | For production: re-select chillers at 2× 18–20 TR scroll/screw with N+1, IPLV-optimised for actual operating regime. Single-compressor reciprocating/scroll handles 25% PLR; centrifugal surge risk at deep part-load. | Spike keeps 2× 50 TR for ML kW signal range. Decision note explicitly distinguishes "spike scenario sizing" vs "production sizing for real loads." |
| **Lag chiller stages by absolute setpoint dwell, but won't fire at 51% load** — real production wants demand-based staging (kW load > X% of lead capacity) in addition to supply-temp fallback | Production staging logic = part-load demand projection + supply-temp safety net. Spike uses supply-temp only (simpler model, sufficient for stage 1-3 demonstration). | Round-2 candidate: add demand-based stage-on (e.g., LOAD_KW > 0.7 × designKW × CH_01_RUN). |
| **Vendor portability across CODESYS-flavored controllers** (WAGO / Beckhoff / Schneider) | The IEC 61131-3 portability claim is **only as good as the I/O config layer** — `%MX10.0` style addressing is target-specific. Pure-logic POUs port; I/O mapping does not. | Add to spike test plan (round 2): export project, attempt re-target to WAGO CODESYS profile, count manual fixes. |
| **Acoustic / structural** (John's check item 11): chiller plant room location not specified in spike, but for real production: spring isolators + inertia base + > NC-50 plant room separation from occupied space | Out of scope for spike (no physical plant). Decision note flags as "production planning item, not stack-comparison signal." | — |

### 5. Verdict — when to use which

> _The actionable section. Fill last after all data in._

```
ESP32 + Node-RED + DB + Power BI ……… choose when:
  • <criterion 1 — e.g., < 10 sensors, no determinism need>
  • <criterion 2 — e.g., cloud-first analytics, business audience>
  • <criterion 3 — e.g., bespoke firmware + custom protocol>

CODESYS + WebVisu + Timescale + Grafana … choose when:
  • <criterion 1 — e.g., > 1 control loop, hard determinism>
  • <criterion 2 — e.g., HMI is operator-facing, not exec-facing>
  • <criterion 3 — e.g., portability across PLC vendors matters>

Crossover indicators — lean CODESYS:
  • _"if you are about to write your own scan loop, stop"_
  • _< more from observation >_

Crossover indicators — lean ESP32:
  • _"if a board can fit the whole system, PLC is overkill"_
  • _< more from observation >_
```

## Open questions surfaced by spike

> _Anything we hit that wasn't in research note #2. Bullet form. Examples we might encounter:_
> - Does Control Win SL trial expire mid-spike? Renewable?
> - Does WebVisu need separate license for production?
> - Does ML IEC Library output expose enough internals to retrain online vs only offline?

## What's next (round-2 candidates, if verdict warrants)

- CODESYS MCP Server hook-up — Atlas/Forge generate POU code via LLM (research note #2 referenced this as Product of the Year 2026)
- Power BI parity test — ingest the same Timescale data into Power BI, compare effort vs Grafana
- Real PLC port — does the same project compile to WAGO PFC200 unchanged?
- HA / multi-site

## References

- Spike brief: [atlas-oracle#3](https://github.com/flukeworachai/atlas-oracle/issues/3)
- Research note: [atlas-oracle#2](https://github.com/flukeworachai/atlas-oracle/issues/2)
- System design: [atlas-oracle#4](https://github.com/flukeworachai/atlas-oracle/issues/4)
- Source design files: `atlas-oracle/ψ/learn/factory-chiller-plant/`
- Setup friction log: `forge-oracle/ψ/lab/codesys-chiller-spike/setup-friction.md`
- POU pseudocode: `forge-oracle/ψ/lab/codesys-chiller-spike/pou-pseudocode.md`
- Math model: `forge-oracle/ψ/lab/codesys-chiller-spike/math-model.md`

— Forge 🔥
