# Verify Windows install state from WSL via `/mnt/c/` — don't ask fluke

**Date**: 2026-05-13
**Source**: CODESYS chiller spike, Batch 2 verification (atlas-oracle#3)
**Context**: Forge lives in WSL2, but guides fluke through Windows-host install steps. Naïve loop: Forge instructs → fluke clicks → fluke describes result → Forge interprets → next instruction. High latency, high translation-loss.

## Pattern

When fluke is installing/configuring software on the Windows host, **read the Windows filesystem from WSL** (`/mnt/c/...`) before asking fluke to describe state. Useful targets:

- `/mnt/c/Users/<user>/AppData/Roaming/<vendor>/` — config dirs, version markers, plugin lists
- `/mnt/c/Users/<user>/AppData/Local/<vendor>/` — caches, install footprints
- `/mnt/c/Program Files/<vendor>/` and `/mnt/c/Program Files (x86)/<vendor>/` — installed binaries, dll names, version subdirs
- `/mnt/c/ProgramData/<vendor>/` — system-wide config, license files
- Windows services list: `powershell.exe -Command "Get-Service <name>*"` from WSL — proves whether install registered services

## Why it works

- File timestamps reveal install order (e.g. AppData/Roaming/CODESYS populated at 10:47 → IDE has launched at least once)
- DLL names reveal capabilities baked in (e.g. `CmpBACnet.dll` → native BACnet support, no external gateway needed → validates John's OQ-4 answer)
- Subdirectory structure reveals install completeness (e.g. `GatewayPLC/AppDataFiles/CODESYSControlWinV3x64/targetvisuextern.cfg` → WebVisu installed)
- Service registration reveals "Complete setup" vs "Custom setup" — Complete bundles everything; Custom requires Package Manager UI step

## Example (CODESYS Batch 2, 11:37)

Pass-1 instructions said: "After Dev System install, open CODESYS Installer (Tools menu) → install Control Win SL + Visualization." Without WSL verification, Forge would have walked fluke through that 5-minute ceremony. With WSL verification:

```bash
ls /mnt/c/ProgramData/CODESYS/                              # → 3 services already
ls "/mnt/c/Program Files/CODESYS 3.5.22.10/"                # → CmpBACnet.dll present
ls "/mnt/c/Users/flukw/AppData/Roaming/CODESYS/"            # → IDE launched 10:47
```

Three commands → "Complete setup bundled everything, skip Batch 2 ceremony, jump straight to Gate 1." Saved ~10 min of click-through and gave Forge evidence (file timestamps, DLL names) to update decision-note §1 with conditional-friction language.

## When NOT to use

- When fluke is interacting with the Windows GUI (clicking through wizards, agreeing to license). File-system polling doesn't capture click-state, dialog focus, or human-in-the-loop choices.
- When the install writes to a path WSL can't see (rare, but e.g. Linux-side Windows Sandbox containers).
- When mid-install — files may be partially written. Verify after a "Setup complete" signal, not during.

## Anti-pattern

Asking fluke "did Control Win SL install yet?" when Forge could have run `ls /mnt/c/ProgramData/CODESYS/` and answered itself. The principle: **observe directly when you can; ask only when you must.**

## Related

- [[spike_codesys_chiller]] — the spike where this pattern emerged
- Pre-flight instructions should verify vendor ship-state from the Store page, not from memory of older versions (sibling lesson — see retro 2026-05/13/23.47)
