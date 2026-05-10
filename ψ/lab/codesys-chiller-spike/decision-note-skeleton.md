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
