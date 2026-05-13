# Setup friction log — CODESYS Chiller Spike

Running log. Each entry: timestamp + step + observation + impact on decision note.

Format:
```
## YYYY-MM-DD HH:MM — <step name>
**Observed**: <what happened — clicks, errors, wait time, surprise>
**Impact**: <how this changes Forge's recommendation vs ESP32 stack>
```

---

## 2026-05-11 09:35 — Download (batch 1)

**Observed**: CODESYS Store delivers **one bundled installer** (`CODESYS 64 3.5.22.10.exe`, 1.4 GB) — not 3 separate .exe files as pass-1 batch-1 instructions anticipated. Control Win SL + Visualization arrive as **packages** installed via the CODESYS Installer UI (Tools menu) *after* the Dev System is on disk.

**Impact**: Modern CODESYS V3.5.16+ uses Package Manager pattern (similar to VS Code extensions). Setup friction at the download stage is **lower** than ESP32 path (1 file, single Store account). Friction shifts to **package management UI** after install — must navigate Installer to add SL + Visu. Decision note §1 (setup friction) should reflect: download = easy, package add = small extra step.

**Plan adjustment**: Batch 2 covers IDE install + first launch + Package Manager add for Control Win SL + Visualization (instead of installing 3 separate downloads). Same end state.

## 2026-05-11 11:37 — Install state (post-Batch-2 verification via /mnt/c/)

**Observed**: Verified state from WSL2 — Complete setup of CODESYS V3.5.22.10 bundled **everything** that the Package Manager step was supposed to add:
- 3 Windows services already registered: `CODESYS Control Win V3 - x64`, `CODESYS Gateway V3`, `CODESYS ServiceControl` (all at 3.5.22.10)
- Runtime binaries present at `GatewayPLC/AppDataFiles/CODESYSControlWinV3x64/` including `targetvisuextern.cfg` (WebVisu support)
- `CmpBACnet.dll` confirms native BACnet support (validates John's OQ-4 answer feasibility — no external gateway needed)
- IDE has launched at least once (AppData/Roaming/CODESYS populated at 10:47)

**Impact**: Batch 2 "open CODESYS Installer → install Control Win SL → install Visualization" steps are **skippable on Complete setup**. The Package Manager UI path documented in pass-1 instructions is a remnant of older CODESYS versions or Custom-setup paths. For Complete setup users, Gate 1 starts directly at "start runtime + create project."

**Decision note input** (§1 Setup friction): the Package Manager UI step adds nominal complexity to setup but is **conditional** — Complete setup avoids it. Decision matrix should note "depends on setup type chosen during installer" rather than as an unconditional friction item.

**Plan adjustment**: Batch 3 sent skipping Step 4-5 ceremony. Direct path: SysTray runtime start → Standard Project → GVL paste → minimal PRG → compile/login/run → Visualization + WebVisu → browser test at `localhost:8080/webvisu.htm` = Gate 1 closed.
