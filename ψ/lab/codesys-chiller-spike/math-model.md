# Chiller plant math model — sim equations

> Why we need a model: spike has no real sensors. Sensor tags must move with believable dynamics so WebVisu trends look alive and ML lib has signal to learn from.
> Goal: minimum equations, realistic *enough* (not first-principles HVAC engineering). Tune-and-forget.

## Design point

| Quantity | Value | Source |
|---------|-------|--------|
| Chiller rated electrical | 250 kW each | spec for FB_Chiller.designKW |
| Chiller rated thermal (RT × 3.517) | ~875 kW thermal | derived from kW_e × COP |
| Design COP (chilled water + warm cond) | 5.0 | mid-range water-cooled centrifugal |
| Design ΔT (T_ret − T_sup) | 5 K | typical primary loop |
| Design CHWS_SP | 7.0 °C | tags.csv default |
| Design ambient (wet-bulb proxy) | 32 °C | Bangkok summer |
| Pump design flow | 30 L/s | matches 875 kW / (4.18 × 5 × 1000) ≈ 41 L/s — round down for spike |
| Pump design ΔP | 150 kPa | typical primary header |
| Scan period (dt) | 1.0 s | PRG_MAIN tick |

## §1. FB_ChillerModel — the heart

Three outputs: `t_sup`, `t_ret`, `kw`. Three inputs that matter: `run`, `chws_sp`, `ambient_t`, `load_share`.

### 1.1 Supply temperature — first-order toward target

```
when run:
    target_sup = chws_sp                         (chiller controls to setpoint)
when not run:
    target_sup = t_ret_last                       (no compressor → drifts up to load)

t_sup(t+dt) = t_sup(t) + (target_sup − t_sup(t)) * (dt / tau_sup_s)

defaults: tau_sup_s = 60   (water mass + heat exchanger thermal lag)
```

Result: when chiller starts cold-side at 12 °C and target is 7 °C, supply reaches ~63% of the way (≈8.8 °C) after one tau, ~95% after 3·tau (3 min). Operators see a believable cool-down trend.

### 1.2 Return temperature — load-driven ΔT

```
when run:
    delta_t = delta_design * load_share * (LOAD_KW / designKW_thermal)
              clipped to [0.5, 8.0]
    t_ret = t_sup + delta_t
when not run:
    t_ret(t+dt) = t_ret(t) + (ambient_t − t_ret(t)) * (dt / tau_ret_off_s)
                  // long decay toward ambient when chiller is off, hours

defaults: delta_design = 5.0 K, tau_ret_off_s = 1800 (30 min)
```

### 1.3 Electrical kW — load + ambient + setpoint

Real chiller power follows compressor map. Simplified:

```
load_pct = load_share * LOAD_KW / designKW_thermal     // 0..1+
                                                         // > 1 means overload (let it spike)

base_factor   = 0.20                                    // baseline draw (auxiliaries + ramp)
slope_factor  = 0.80                                    // load-proportional component

# COP degrades as condenser water gets warm (ambient proxy)
cop_factor = 1.0 + 0.04 * (ambient_t − 32.0)           // +4% kW per K above 32

# Setpoint penalty: lower setpoint = more work
sp_penalty = 1.0 + 0.025 * (7.0 − chws_sp)             // +2.5% kW per K below 7

kw = designKW * (base_factor + slope_factor * load_pct) * cop_factor * sp_penalty

# clamp
kw = max(0.0, min(kw, 1.5 * designKW))                 // hard ceiling to keep range realistic
```

At design (load_pct=1, ambient=32, sp=7): kw = 250 × (0.2 + 0.8) × 1.0 × 1.0 = **250 kW** ✓.
At part load (load_pct=0.4): kw = 250 × (0.2 + 0.32) = 130 kW (~52% of design draw at 40% load — non-linear, matches real machines).

### 1.4 Anomaly hook (stage 3)

Add input `kw_drift_pct : REAL := 0.0;` and modify last line:

```
kw = kw * (1.0 + kw_drift_pct / 100.0)
```

Stage 3 ramps `kw_drift_pct` from 0 → 15 over an hour at constant load to simulate fouling/refrigerant degradation. ML lib trained on baseline kW(load) curve flags the deviation.

## §2. FB_Pump — trivial constant-speed

| State | flow | dp | kw |
|-------|------|----|----|
| run = TRUE | 30 L/s | 150 kPa | 7.5 kW |
| run = FALSE | 0 | 0 | 0 |

If we want VFD later: `flow = DESIGN_FLOW × speed_pct`, `kw ∝ speed^3` (affinity laws). Out of spike scope.

## §3. FB_CoolingTower

### 3.1 Basin temperature — first-order toward target

```
heat_to_reject_kw = (CH_01_KW + CH_02_KW) * (1 + 1/COP_design)
                  ≈ (electrical input) + (thermal removed)
                  = (CH_01_KW + CH_02_KW) * 1.25 if COP=5

if fan_run:
    target = ambient_t + 4.0 + heat_to_reject * 0.005    // 4 K approach + load rise
else:
    target = ambient_t + 8.0 + heat_to_reject * 0.010    // worse approach without forced air

basin_t(t+dt) = basin_t(t) + (target − basin_t(t)) * (dt / 120)
```

### 3.2 Fan control — simple hysteresis

```
if basin_t > 32 °C: fan_run = TRUE
if basin_t < 26 °C: fan_run = FALSE
```

(Real towers use VFD on fan with PID against basin or condenser temp — keep ON/OFF for spike.)

### 3.3 Makeup valve

Crude proxy for evaporation rate:

```
makeup_pct = clip(0..100, heat_to_reject * 0.05)        // larger heat → more evap → more makeup
                                                         // 250 kW load → ~12.5 % open
```

Real plants use level switch in basin + drift/blowdown calc. Stage-2 fidelity not required.

## §4. Ambient & load profiles (driving signals)

For stage 2 baseline, leave:
- `AMBIENT_T = 32.0` (constant; can be a slider in WebVisu)
- `LOAD_KW = 350.0` (constant; ~40% of single-chiller design = lead-only stable)

For stage 3 baseline (clean run for ML training):
- `LOAD_KW` follows daily cycle: `350 + 200 * sin(2π t / 86400)` clipped at 600 kW
- `AMBIENT_T` follows `30 + 4 * sin(2π t / 86400 − π/4)` (lags load slightly)
- Run for ~24 sim minutes (compressed time = 86400/24/60 ≈ 60× — `dt = 60s` virtual per `1s` real)

For stage 3 anomaly run:
- Same profile + `kw_drift_pct` ramp 0 → 15 over 1 sim hour starting at sim t = 30 min

## §5. Sanity-check expected ranges (for tags.csv bounds)

| Tag | Bound (csv) | Sim range observed | OK? |
|-----|-------------|--------------------|-----|
| CH_xx_T_SUP | -5..20 | 5..14 (warm-up to running) | ✓ |
| CH_xx_T_RET | 5..30 | 7..22 | ✓ |
| CH_xx_KW | 0..500 | 0..380 (single, w/ drift) | ✓ |
| PMP_01_FLOW | 0..50 | 0 or 30 | ✓ |
| PMP_01_KW | 0..30 | 0 or 7.5 | ✓ |
| CT_01_BASIN_T | 10..40 | 26..36 | ✓ |
| CT_01_MAKEUP_VLV | 0..100 | 0..30 | ✓ |

If any tag pegs the bound, widen the bound rather than clamping the model — bounds are display hints, not control.

## §6. Test plan (gate-level)

- **Gate 1 sanity** (post-WebVisu): with `LOAD_KW = 350`, `SCHED_ON = TRUE`, `LEADLAG_MODE = 0` — expect within 5 min: CH_01 RUN, T_SUP → 7 °C, KW ~140, BASIN_T → ~32 °C with fan toggling.
- **Gate 1 step test**: bump `LOAD_KW` to 700 → expect T_SUP rise > 9 °C → after 5 min lag_timer fires → CH_02 starts → T_SUP recovers.
- **Gate 1 fault test**: force `CH_01_ALM := TRUE` → expect lead transfer to CH_02, `PLANT_ALM_ANY = TRUE`.
- **Gate 3 anomaly**: clean baseline run + drift run → ML lib classifier separates them at kw_drift > ~5%.

— Forge 🔥
