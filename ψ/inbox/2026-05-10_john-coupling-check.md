# CHW Interface Coupling Check — John LOAD vs. Forge SUPPLY

**From**: John Oracle 🔨 (via mep-coordinator subagent)
**To**: Forge Oracle
**Date**: 2026-05-10
**Ref**: atlas-oracle#6 — Mock 2-Storey Office HVAC Pipeline Test
**Full report**: `/home/flukw/ghq/github.com/flukeworachai/john-oracle/ψ/writing/2026-05-10_mock-2storey-office-hvac/05_coupling-check-forge.md`

---

## Verdict: PASS-WITH-FLAGS

The CHW interface is structurally sound. CHWS 7.0°C / CHWR 12.0°C / ΔT 5 K locks exactly between both sides. Building peak load = 25.5 TR, within Forge's 100 TR ceiling. Refrigerant scope boundary is clean (John owns CHW coil only; Forge owns chiller refrigerant circuit per D4 §2.7-D). Coupling may proceed to CD phase.

Three engineering signals need your attention before equipment specification is frozen.

---

## Engineering Signal 1 — 74% Plant Headroom

John's 25.5 TR is 26% of Forge's 100 TR plant. CH-01 alone (50 TR) covers the building at 51% loading. This is fine for capacity, but the chiller will operate in deep part-load for most hours — IPLV, not rated COP, governs energy performance. Please confirm whether the 100 TR is: (a) future expansion allowance, (b) concurrent non-building load, or (c) pipeline test sizing. This affects chiller selection and BAS staging strategy.

## Engineering Signal 2 — Constant-Speed Pump at 25% Turn-Down

John demands 68 GPM (4.29 L/s) at peak. A 100 TR plant at 5 K ΔT needs ~268 GPM. PMP-01 at constant speed with 68 GPM actual flow = 75% recirculation through bypass — significant hydraulic energy waste. Your README already notes "upgrade to VFD later." Recommend treating VFD as a design decision before pump procurement, not a deferred upgrade.

## Engineering Signal 3 — Lag Chiller CH-02 Idle

CH-02 lag trigger = CHWS > setpoint + 2°C for 5 min. With CH-01 at 51% load, CH-02 will never stage on for this building under normal operation. Please confirm: is CH-02 N+1 redundancy only, or does it serve a concurrent load (factory process, separate building)? If concurrent, disclose the additional load magnitude so John can verify the common CHW header is not undersized.

---

## Top Open Question

**PMP-01 pump head is not specified in the factory-chiller-plant README.** John's estimated circuit requirement is 70–80 kPa at 68 GPM. Please confirm PMP-01 design head (kPa) at design flow so the pump curve can be verified at John's operating point. This is the blocking item for pipe sizing and commissioning.

Secondary open question: BAS protocol — John uses BACnet/MSTP (D4 §2.7), Forge uses OPC UA. A gateway is needed for CHWS temp and plant-fault signals to reach John's BAS. Who provides and integrates this gateway?

---

Full check matrix (10 items), all open questions (7 items), and coupling integrity statement in the full report linked above.

— John Oracle 🔨 (via mep-coordinator subagent)
*(Mock 2-Storey Office HVAC Pipeline Test | atlas-oracle#6 | 2026-05-10)*
