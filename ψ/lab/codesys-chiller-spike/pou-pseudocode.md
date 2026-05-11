# POU + FB pseudocode — Chiller Plant Spike

> IEC 61131-3 Structured Text (ST) style. **Pass-2 syntax-compliant** for CODESYS V3 Dev System.
> Tag names mirror `atlas-oracle/ψ/learn/factory-chiller-plant/tags.csv` plus 2 CHWR alarm tags per John's OQ-7 at [atlas-oracle#6](https://github.com/flukeworachai/atlas-oracle/issues/6) — **27 tags total** (was 25).
> Math model lives in `FB_ChillerModel`, delegated by `FB_Chiller` — see `math-model.md` for equations.

## Architecture

```
PRG_MAIN (1 s scan)
  ├─ ReadInputs            (no real I/O — direct GVL_TAGS access)
  ├─ fbAlarmMgr.Run()      (clears stale, latches new)
  ├─ fbLeadLag.Run()       (decides which chiller is lead)
  ├─ fbCh01.Run() / fbCh02.Run()
  │     └─ fbModel.Step()  (math model — sets T_SUP, T_RET, KW)
  ├─ fbPump.Run()
  ├─ fbCT.Run()
  └─ WriteOutputs          (publish back to GVL_TAGS)
```

## GVL_TAGS

Direct mapping from `tags.csv`. CODESYS accepts CSV import; if not, paste this block.

```iecst
VAR_GLOBAL
    // ── Chiller 1 ──
    CH_01_RUN     : BOOL;
    CH_01_ALM     : BOOL;
    CH_01_T_SUP   : REAL;   // -5..20 degC
    CH_01_T_RET   : REAL;   //  5..30 degC
    CH_01_KW      : REAL;   //  0..500 kW
    CH_01_RUNHRS  : DINT;
    CH_01_LEAD    : BOOL;
    CH_01_T_RET_HI_ALM : BOOL;   // T_RET > 14°C for 2 min (per John OQ-7, atlas#6)

    // ── Chiller 2 ──
    CH_02_RUN     : BOOL;
    CH_02_ALM     : BOOL;
    CH_02_T_SUP   : REAL;
    CH_02_T_RET   : REAL;
    CH_02_KW      : REAL;
    CH_02_RUNHRS  : DINT;
    CH_02_LEAD    : BOOL;
    CH_02_T_RET_HI_ALM : BOOL;   // T_RET > 14°C for 2 min (per John OQ-7, atlas#6)

    // ── Pump ──
    PMP_01_RUN    : BOOL;
    PMP_01_KW     : REAL;
    PMP_01_FLOW   : REAL;   // L/s
    PMP_01_DP     : REAL;   // kPa

    // ── Cooling Tower ──
    CT_01_FAN_RUN     : BOOL;
    CT_01_BASIN_T     : REAL;   // 10..40 degC
    CT_01_MAKEUP_VLV  : REAL;   // 0..100 %

    // ── Setpoints / Modes ──
    CHWS_SP       : REAL := 7.0;    // chilled water supply setpoint, degC
    LEADLAG_MODE  : DINT := 0;      // 0=auto, 1=force CH1 lead, 2=force CH2 lead
    SCHED_ON      : BOOL := TRUE;

    // ── Plant alarm summary ──
    PLANT_ALM_ANY : BOOL;
END_VAR
```

Internal-only (not in tags.csv but needed for sim):

```iecst
VAR_GLOBAL
    AMBIENT_T     : REAL := 32.0;   // outdoor / wet-bulb proxy, degC
    LOAD_KW       : REAL := 350.0;  // total cooling load demand, kW thermal
    LOAD_PROFILE_T : TIME := T#0s;   // for ramped load testing
END_VAR
```

## PRG_MAIN

```iecst
PROGRAM PRG_MAIN
VAR
    // FB instances
    fbCh01      : FB_Chiller;
    fbCh02      : FB_Chiller;
    fbLeadLag   : FB_LeadLag;
    fbPump      : FB_Pump;
    fbCT        : FB_CoolingTower;
    fbAlarmMgr  : FB_AlarmManager;

    // scan tick
    cycleTimer  : TON;
    initDone    : BOOL := FALSE;
END_VAR

// ── one-shot init ──
IF NOT initDone THEN
    fbCh01.id    := 1;
    fbCh02.id    := 2;
    fbCh01.designKW := 250.0;   // RT-rated electrical at full load
    fbCh02.designKW := 250.0;
    initDone := TRUE;
END_IF

// ── alarm manager runs first (latch state for everyone) ──
// CHWR alarms read previous-cycle GVL values — 1-tick (1 s) lag is negligible
// against the 2-min CHWR dwell timer inside FB_Chiller.
fbAlarmMgr.ch01_alm     := CH_01_ALM;
fbAlarmMgr.ch02_alm     := CH_02_ALM;
fbAlarmMgr.ch01_chwr_hi := CH_01_T_RET_HI_ALM;
fbAlarmMgr.ch02_chwr_hi := CH_02_T_RET_HI_ALM;
fbAlarmMgr();
PLANT_ALM_ANY := fbAlarmMgr.anyActive;

// ── lead/lag selection ──
fbLeadLag.mode      := LEADLAG_MODE;
fbLeadLag.ch01_hrs  := CH_01_RUNHRS;
fbLeadLag.ch02_hrs  := CH_02_RUNHRS;
fbLeadLag.ch01_alm  := CH_01_ALM;
fbLeadLag.ch02_alm  := CH_02_ALM;
fbLeadLag.t_sup     := CH_01_T_SUP;       // use CH-01 reading as plant supply (or PMP discharge later)
fbLeadLag.chws_sp   := CHWS_SP;
fbLeadLag();
CH_01_LEAD := fbLeadLag.ch01_lead;
CH_02_LEAD := fbLeadLag.ch02_lead;

// ── chiller 1 ──
fbCh01.lead         := CH_01_LEAD;
fbCh01.lag          := fbLeadLag.ch01_lag;
fbCh01.sched_on     := SCHED_ON;
fbCh01.chws_sp      := CHWS_SP;
fbCh01.ambient_t    := AMBIENT_T;
fbCh01.load_share   := fbLeadLag.ch01_load_share;
fbCh01();
CH_01_RUN          := fbCh01.run;
CH_01_T_SUP        := fbCh01.t_sup;
CH_01_T_RET        := fbCh01.t_ret;
CH_01_KW           := fbCh01.kw;
CH_01_RUNHRS       := fbCh01.runhrs;
CH_01_T_RET_HI_ALM := fbCh01.chwr_hi_alarm;

// ── chiller 2 (mirrors CH_01 wiring) ──
fbCh02.lead         := CH_02_LEAD;
fbCh02.lag          := fbLeadLag.ch02_lag;
fbCh02.sched_on     := SCHED_ON;
fbCh02.chws_sp      := CHWS_SP;
fbCh02.ambient_t    := AMBIENT_T;
fbCh02.load_share   := fbLeadLag.ch02_load_share;
fbCh02();
CH_02_RUN          := fbCh02.run;
CH_02_T_SUP        := fbCh02.t_sup;
CH_02_T_RET        := fbCh02.t_ret;
CH_02_KW           := fbCh02.kw;
CH_02_RUNHRS       := fbCh02.runhrs;
CH_02_T_RET_HI_ALM := fbCh02.chwr_hi_alarm;

// ── pump ──
fbPump.sched_on     := SCHED_ON;
fbPump.any_chiller_run := CH_01_RUN OR CH_02_RUN;
fbPump();
PMP_01_RUN    := fbPump.run;
PMP_01_KW     := fbPump.kw;
PMP_01_FLOW   := fbPump.flow;
PMP_01_DP     := fbPump.dp;

// ── cooling tower ──
fbCT.heat_rejection_kw := CH_01_KW + CH_02_KW + (CH_01_KW + CH_02_KW) * 1.25;  // electrical + thermal removed
fbCT.ambient_t  := AMBIENT_T;
fbCT();
CT_01_FAN_RUN    := fbCT.fan_run;
CT_01_BASIN_T    := fbCT.basin_t;
CT_01_MAKEUP_VLV := fbCT.makeup_pct;

END_PROGRAM
```

## FB_Chiller

```iecst
FUNCTION_BLOCK FB_Chiller
VAR_INPUT
    id                : INT;       // 1 or 2 (for diagnostics)
    lead              : BOOL;      // assigned lead
    lag               : BOOL;      // currently committed as lag
    sched_on          : BOOL;
    chws_sp           : REAL;
    ambient_t         : REAL;
    load_share        : REAL;      // 0..1 fraction of plant load this chiller serves
    designKW          : REAL := 250.0;
    chwr_hi_threshold : REAL := 14.0;     // CHWR alarm threshold, °C (John OQ-7)
    chwr_hi_dwell     : TIME := T#2m;     // dwell before latch
END_VAR
VAR_OUTPUT
    run               : BOOL;
    t_sup             : REAL := 12.0;     // start warm
    t_ret             : REAL := 17.0;
    kw                : REAL;
    runhrs            : DINT;
    healthy           : BOOL := TRUE;
    chwr_hi_alarm     : BOOL;             // latches when t_ret > threshold for dwell time
END_VAR
VAR
    fbModel           : FB_ChillerModel;
    chwrHiTimer       : TON;
    runhrs_ms         : ULINT;            // accumulator before /3600000 → DINT
    last_run          : BOOL;
END_VAR

// ── start/stop logic ──
run := sched_on AND healthy AND (lead OR lag);

// ── runtime accumulation ──
IF run THEN
    runhrs_ms := runhrs_ms + 1000;   // 1 s scan
END_IF
runhrs := DINT(runhrs_ms / 3600000);

// ── delegate to math model ──
fbModel.run        := run;
fbModel.chws_sp    := chws_sp;
fbModel.ambient_t  := ambient_t;
fbModel.load_share := load_share;
fbModel.designKW   := designKW;
fbModel.dt_s       := 1.0;
fbModel();

t_sup := fbModel.t_sup;
t_ret := fbModel.t_ret;
kw    := fbModel.kw;

// ── CHWR high alarm (John OQ-7, atlas#6 — diagnostic, does not trip chiller) ──
chwrHiTimer(IN := (t_ret > chwr_hi_threshold), PT := chwr_hi_dwell);
chwr_hi_alarm := chwrHiTimer.Q;

last_run := run;

// ── health (placeholder — extend with high-temp/low-flow trips later) ──
// healthy is set externally for now (alarm injection); FB self-faults reserved for round 2.

END_FUNCTION_BLOCK


// helper FB — see math-model.md for equations
FUNCTION_BLOCK FB_ChillerModel
VAR_INPUT
    run         : BOOL;
    chws_sp     : REAL;
    ambient_t   : REAL;
    load_share  : REAL;
    designKW    : REAL;
    dt_s        : REAL := 1.0;
END_VAR
VAR_OUTPUT
    t_sup       : REAL := 12.0;
    t_ret       : REAL := 17.0;
    kw          : REAL;
END_VAR
VAR
    tau_sup_s   : REAL := 60.0;     // first-order time constant
    delta_design : REAL := 5.0;     // ΔT at design load (K)
    cop_design   : REAL := 5.0;     // electrical efficiency at design conditions
END_VAR

// see math-model.md §1 — pseudocode body filled there
END_FUNCTION_BLOCK
```

## FB_LeadLag

```iecst
FUNCTION_BLOCK FB_LeadLag
VAR_INPUT
    mode        : DINT;      // 0=auto, 1=force CH1 lead, 2=force CH2 lead
    ch01_hrs    : DINT;
    ch02_hrs    : DINT;
    ch01_alm    : BOOL;
    ch02_alm    : BOOL;
    t_sup       : REAL;
    chws_sp     : REAL;
END_VAR
VAR_OUTPUT
    ch01_lead   : BOOL;
    ch02_lead   : BOOL;
    ch01_lag    : BOOL;
    ch02_lag    : BOOL;
    ch01_load_share : REAL;
    ch02_load_share : REAL;
END_VAR
VAR
    lagOnTimer  : TON;
    rotationDue : BOOL;
    activeLead  : INT := 1;       // 1 or 2
END_VAR

// ── pick lead by mode ──
CASE mode OF
    1: activeLead := 1;
    2: activeLead := 2;
    0:
        // rotate by run hours every 168 hr OR fault on current lead
        rotationDue := ABS(ch01_hrs - ch02_hrs) > 168;
        IF activeLead = 1 AND (ch01_alm OR rotationDue) THEN
            IF NOT ch02_alm THEN activeLead := 2; END_IF
        ELSIF activeLead = 2 AND (ch02_alm OR rotationDue) THEN
            IF NOT ch01_alm THEN activeLead := 1; END_IF
        END_IF
END_CASE

ch01_lead := (activeLead = 1) AND NOT ch01_alm;
ch02_lead := (activeLead = 2) AND NOT ch02_alm;

// ── lag commit when supply > sp + 2°C for 5 min ──
lagOnTimer(IN := (t_sup > chws_sp + 2.0), PT := T#5m);

ch01_lag := (activeLead = 2) AND lagOnTimer.Q AND NOT ch01_alm;
ch02_lag := (activeLead = 1) AND lagOnTimer.Q AND NOT ch02_alm;

// ── load share ──
IF (ch01_lead OR ch01_lag) AND (ch02_lead OR ch02_lag) THEN
    ch01_load_share := 0.5;
    ch02_load_share := 0.5;
ELSE
    ch01_load_share := SEL(ch01_lead OR ch01_lag, 0.0, 1.0);
    ch02_load_share := SEL(ch02_lead OR ch02_lag, 0.0, 1.0);
END_IF

END_FUNCTION_BLOCK
```

## FB_Pump

```iecst
FUNCTION_BLOCK FB_Pump
VAR_INPUT
    sched_on        : BOOL;
    any_chiller_run : BOOL;
END_VAR
VAR_OUTPUT
    run             : BOOL;
    kw              : REAL;
    flow            : REAL;
    dp              : REAL;
END_VAR
VAR CONSTANT
    // ── Spike sizing — plant-scale (factory chiller scenario, 100 TR ceiling). ──
    // For real building-coupled design (per atlas#6, 25.5 TR office):
    //   resize to ~5.4 L/s (85 GPM) @ 90 kPa, ~2 kW VFD primary.
    // ASHRAE 90.1-2022 §6.5.4.2 mandates VFD on variable-flow primaries > 7.5 kW.
    // Captured in decision-note-skeleton.md §4 Deployment.
    DESIGN_FLOW_LPS : REAL := 30.0;   // 475 GPM @ spike — building-coupled ≈ 5.4 L/s
    DESIGN_DP_KPA   : REAL := 150.0;  // spike — building-coupled ≈ 90 kPa
    DESIGN_KW       : REAL := 7.5;    // spike — building-coupled ≈ 2 kW with VFD
END_VAR

// constant-speed pump — runs whenever chillers are committed
run := sched_on AND any_chiller_run;

IF run THEN
    flow := DESIGN_FLOW_LPS;
    dp   := DESIGN_DP_KPA;
    kw   := DESIGN_KW;
ELSE
    flow := 0.0;
    dp   := 0.0;
    kw   := 0.0;
END_IF

END_FUNCTION_BLOCK
```

## FB_CoolingTower

```iecst
FUNCTION_BLOCK FB_CoolingTower
VAR_INPUT
    heat_rejection_kw : REAL;     // total heat to reject (cond side)
    ambient_t   : REAL;
END_VAR
VAR_OUTPUT
    fan_run     : BOOL;
    basin_t     : REAL := 28.0;
    makeup_pct  : REAL;
END_VAR
VAR CONSTANT
    BASIN_HI_T  : REAL := 32.0;
    BASIN_LO_T  : REAL := 26.0;
    APPROACH_K  : REAL := 4.0;     // wet-bulb approach when fan on
    DT_S        : REAL := 1.0;
    TAU_S       : REAL := 120.0;   // basin thermal time constant
END_VAR
VAR
    target  : REAL;                // first-order target (declared at top — pass-2 fix)
END_VAR

// fan hysteresis
IF basin_t > BASIN_HI_T THEN fan_run := TRUE; END_IF
IF basin_t < BASIN_LO_T THEN fan_run := FALSE; END_IF

// basin temperature first-order
IF fan_run THEN
    target := ambient_t + APPROACH_K + heat_rejection_kw * 0.005;  // rough rise per kW
ELSE
    target := ambient_t + 8.0 + heat_rejection_kw * 0.01;
END_IF
basin_t := basin_t + (target - basin_t) * (DT_S / TAU_S);

// makeup ~ proportional to evaporation (heat / latent)
makeup_pct := SEL(fan_run, 0.0, MIN(100.0, heat_rejection_kw * 0.05));

END_FUNCTION_BLOCK
```

## FB_AlarmManager

```iecst
FUNCTION_BLOCK FB_AlarmManager
VAR_INPUT
    ch01_alm        : BOOL;
    ch02_alm        : BOOL;
    ch01_chwr_hi    : BOOL;       // per John OQ-7, atlas#6
    ch02_chwr_hi    : BOOL;
    pump_fault      : BOOL;
    ct_fault        : BOOL;
END_VAR
VAR_OUTPUT
    anyActive       : BOOL;
    activeCount     : INT;
END_VAR

activeCount := 0;
IF ch01_alm     THEN activeCount := activeCount + 1; END_IF
IF ch02_alm     THEN activeCount := activeCount + 1; END_IF
IF ch01_chwr_hi THEN activeCount := activeCount + 1; END_IF
IF ch02_chwr_hi THEN activeCount := activeCount + 1; END_IF
IF pump_fault   THEN activeCount := activeCount + 1; END_IF
IF ct_fault     THEN activeCount := activeCount + 1; END_IF

anyActive := activeCount > 0;

END_FUNCTION_BLOCK
```

## CODESYS-syntax notes (pass-2 verified)

- **Comments**: `//` single-line and `(* … *)` block comments both work in CODESYS V3 ST. Pass-2 uses `//` throughout.
- **`SEL(cond, IN0, IN1)`**: IEC 61131-3 ternary — returns `IN0` when `cond=FALSE`, `IN1` when `cond=TRUE`. Used in `FB_LeadLag.load_share` and `FB_CoolingTower.makeup_pct`.
- **Variable declarations**: ALL local vars must be in `VAR ... END_VAR` blocks BEFORE the body. `DECLARE` mid-body (used in pass-1) is **NOT** standard IEC — fixed in `FB_CoolingTower.target` (now in a `VAR` section).
- **Type conversions**: explicit `DINT(x)`, `REAL(x)`, etc. — used in `FB_Chiller.runhrs` accumulator.
- **Timer types**: `TON` instance + call form `lagOnTimer(IN := …, PT := T#5m)`. `T#` literal prefix for `TIME` constants.
- **Method blocks**: pseudocode keeps logic in the FB body. If wanting `FB.Start()` / `.Stop()` / `.IsHealthy()` shape, refactor as Methods in CODESYS GUI — identical logic, different surface.
- **GVL access**: `VAR_GLOBAL` accessed by simple name across all POUs once declared. Pass-2 uses unqualified names (`CHWS_SP`, not `GVL_TAGS.CHWS_SP`) — works in V3 if there's no name collision.

## Design intent notes

- **Anomaly injection** for stage 3: `FB_ChillerModel` will accept a `kw_drift_pct : REAL := 0.0;` input (see `math-model.md §1.4`), multiplies `kw` output by `1.0 + drift_pct/100`. Hook reserved; not yet plumbed in PRG_MAIN.
- **Load profile**: `LOAD_KW` in GVL is a free-running variable. Stage 2 = WebVisu slider; stage 3 = sine / random walk driven from PRG.
- **Pump sizing — spike vs real**: `FB_Pump` constants reflect plant-scale (475 GPM @ 150 kPa). Real building-coupled design (per [atlas#6](https://github.com/flukeworachai/atlas-oracle/issues/6) coupling check) = ~85 GPM @ 90 kPa, ~2 kW VFD primary. Captured in `decision-note-skeleton.md §4 Deployment`.
- **CHWR high alarm** (pass-2 new, per John's OQ-7 at atlas#6): `T_RET > 14°C` for 2 min latches `CH_0X_T_RET_HI_ALM`. Folded into `PLANT_ALM_ANY` via `FB_AlarmManager`. **Diagnostic-only** — does not trip chiller. Reading at PRG_MAIN is previous-cycle GVL (1 s lag, negligible vs 2 min dwell).
- **Tag count**: 25 → 27. Upstream `atlas-oracle/ψ/learn/factory-chiller-plant/tags.csv` to be synced by Atlas. Suggested CSV rows:
  ```
  CH-01_T_RET_HI_ALM,CH-01,Chiller,BOOL,,,,CHWR high temp alarm (T_RET>14°C × 2 min),%MX10.3
  CH-02_T_RET_HI_ALM,CH-02,Chiller,BOOL,,,,CHWR high temp alarm (T_RET>14°C × 2 min),%MX11.3
  ```

— Forge 🔥
