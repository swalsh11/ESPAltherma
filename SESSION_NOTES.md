# Session Notes — 2026-08-09

Solar inverter integration, Home Assistant rebuild, and Daikin Altherma solar-diversion research.

## 1. KSpark inverter discovery and Home Assistant integration

- Discovered new KSpark inverter (Solis-badged, model **E6KS-D22**, hybrid with battery) on the LAN at `192.168.0.3`, logger serial `1133961805`, inverter SN `210214495G22209900101`.
- Installed the `home_assistant_solarman` HACS integration (local Modbus over Solarman V5 protocol, port 8899).
- Initial custom register map (`kspark_solis.yaml`) was built by manual probing and was **wrong** for several fields (grid frequency, grid voltage, PV power) — confirmed by cross-checking against the SolarmanPV cloud dashboard (`globalhome.solarmanpv.com`) as ground truth.
- Found the actual correct register map: `kstar_hybrid.yaml` (bundled with the integration) — the unit is built on a KStar hybrid platform. Verified live against cloud data: grid voltage, frequency, battery SoC/temp, system status all matched exactly. Switched the integration to use this definition.
- Retired the old REST sensors that scraped the logger's web UI (`192.168.0.3/status.html`) — proved unreliable/inconsistent for power values.
- Rebuilt `template.yaml` solar sensor chain (`Solar Power`, `Solar Daily Energy`, `KSpark Solar *`) to source from the validated `kstar_hybrid.yaml` entities (PV1+PV2 power, Daily/Cumulative Production).
- Fixed a dead reference: `weather.forecast_home` → `weather.home` (real entity name) in the outdoor-temp fallback template.
- Removed decommissioned RESOL solar-thermal ESPHome integration.
- Configured the HA **Energy Dashboard**: grid import/export from `Cumulative Energy Purchased`/`Cumulative Grid Feed-In`, solar production from `KSpark Solar Total`, battery from `Battery Total Charge`/`Battery Total Discharge`.

### Critical evaluation findings (Solarman remote settings)
- Grid Code: Ireland ✓, protection thresholds standard, anti-islanding **enabled** (safety-critical, correct).
- Found 4 alerts in Solarman's Alerts page (`W00 Grid Volt Low`, `W02 Grid Frequency`, `W05 Bat Loss`, `W20 Bms Communication`), all clustered within a 12-minute window on **2026-08-05** (install day) — read as one commissioning event (BMS comms hiccup cascading to a brief grid dropout), not an ongoing fault. Nothing recurred in the 4 days since.
- Outstanding, not yet actioned: open WiFi AP still broadcasting (`AP_1133961805`) on the logger, and default `admin/admin` logger web credentials — both fixable from the SolarmanPV app.

## 2. Daikin Altherma solar-diverted DHW heating

Goal: heat domestic hot water preferentially when there's solar surplus, without touching space heating.

- Confirmed via the official Daikin installer reference guide (EBLA04~08E2V3+E23V3, covers the user's EBLA06E23V3) that the unit has a genuine **Smart Grid input** — two dry contacts (S10S/S11S) on terminal block **X5M** (X5M.9/10 and X5M.5/6), spec'd at ~16V DC detection supplied by the unit's own PCB (voltage-free/dry contact, do not drive with external voltage — use relay contacts only).
- Truth table: `00`=Free running, `01`=Forced off, `10`=**Recommended on** (buffers PV surplus into DHW tank, capacity-limited to available surplus), `11`=Forced on (no capacity limit).
- Field settings needed on the unit's installer menu: `[9.8.4]`=3 (Smart grid), `[9.8.7]`=No (DHW-only buffering, not room/space heating).
- Source: Daikin installer reference guide `4PEN685228-1B`, my.daikin.eu.

### ESPAltherma firmware (existing hardware, no new device needed)
- User's ESP32 (WT32-ETH01 board, wired Ethernet) already runs ESPAltherma (raomin/ESPAltherma, based on PR #399 for WT32-ETH01 support) for read-only P1P2 bus monitoring — confirmed **read-only**, cannot write arbitrary heat pump config, but **does** support driving relays on the SG1/SG2 dry contacts via `PIN_SG1`/`PIN_SG2` GPIO + MQTT (`espaltherma/sg/set`, values 0-3 matching the Daikin truth table above).
- `setup.h` template already had `PIN_SG1=32`/`PIN_SG2=33` pre-assigned (commented out). Confirmed both are safe, free GPIOs on WT32-ETH01 — not used by the onboard LAN8720 Ethernet PHY (reserves GPIO0/16/18/23) and not boot-strapping pins. **Uncommented both** in the live `setup.h` on the NAS.
- Found and fixed a real bug: SG mode was never persisted to EEPROM, so every reboot silently reset it to mode 0 regardless of what it was set to, and MQTT reconnect always re-announced "0" to HA regardless of truth.
  - Merged the fix from upstream PR **#457** ("Restore SG state after reset or MQTT conn lost") into the live firmware — `mqtt.h` gained `EEPROM_SG` storage, `digitalWriteSgPins()`/`saveSgState()` helpers, and `restoreEEPROM()` (renamed from `readEEPROM()`) now restores SG mode on boot and republishes real state on MQTT reconnect.
  - Confirmed upstream PR **#260** (MQTT discovery improvements) was already effectively present in the live firmware — no work needed there.
  - All new SG code is gated behind `#ifdef PIN_SG1`, so this was a zero-behavior-change merge until the pins were uncommented.
- **Definition files intentionally left untouched** (`include/def/DEFAULT.h` stays as-is). Investigated switching to the closest match, `Altherma(EBLA-EDLA D series 4-8kW Monobloc).h`, but it's for the **D-series**, not the user's **E-series EBLA06E23V3** — no E-series-specific file exists even in current upstream. Critically, it renames 3 labels the existing `template.yaml` depends on for COP/Heat Output calculations (`Outlet PHE(R1T)`, `Inlet temperature(R4T)`, `DHW temperature(R5T)`) — switching would have silently broken COP tracking. Decision: stay on `DEFAULT.h`; if the extra diagnostic fields (Target Evap/Cond Temp, Space heating Operation ON/OFF, Reheat/Storage ECO flags) are wanted later, add them as new lines to the existing file rather than replacing it wholesale.
- Also found (separately, not yet fixed): `template.yaml` references a `Room Setpoint` attribute that doesn't exist in either `DEFAULT.h` or the D-series file — that HA sensor has likely been silently returning its `0` fallback regardless of firmware.
- Fixed a stale `platformio.ini`: `default_envs` was pointing at `espoe32` (leftover from an earlier hardware iteration — PR #401 — before the user switched to WT32-ETH01, not a second live device), switched to `wt32-eth01`. OTA `upload_port` was a hardcoded IP (`192.168.0.21`) that had drifted via DHCP to the device's actual current address (`192.168.0.189`, confirmed live) — switched to the `ESPAltherma.local` mDNS hostname so this doesn't recur.
- Surveyed 3 firmware snapshots on the NAS (`ESPAltherma-main` = live/current, `ESPAltherma-mainx` = Feb 2024 experiments, `ESPAltherma32` = Feb 2023 earliest experiments) — confirmed the two older ones are superseded, nothing uniquely valuable in them, all three were always on `DEFAULT.h`.

### Git / repo state
- Live firmware source lives at `/volume1/Sean/e15 backup/ESPAltherma-main/` on the NAS (not itself a git repo — plain files). `.gitignore` there excludes `src/setup.h` (correctly keeps WiFi/MQTT credentials out of version control).
- Cloned the existing bare remote (`ssh://Admin_sean@192.168.0.112/volume1/git-repos/esp-altherma.git`) to this machine at `~/projects/esp-altherma`, synced in the live firmware content, committed the #457 merge + platformio.ini fixes (commit `63050f2`), pushed to `origin` (NAS, fast-forward).
- `github.com/swalsh11/ESPAltherma` has its own real, unrelated commit history (a fork of upstream `raomin/ESPAltherma` at commit `ebc7757`) — pushing the local single-commit snapshot to `main` there would have required a force-push and destroyed that history, so instead pushed as a new branch: **`wt32-eth01-sg-relay`**. `main` on GitHub is untouched. PR link: https://github.com/swalsh11/ESPAltherma/pull/new/wt32-eth01-sg-relay

### Outstanding / next steps
1. Wire two relays: GPIO32 → relay → S10S (X5M.9/10), GPIO33 → relay → S11S (X5M.5/6) on the Daikin outdoor unit.
2. Build the `wt32-eth01` PlatformIO environment and flash (OTA now works via `ESPAltherma.local`, no need to remove the board).
3. On the Daikin installer menu: set `[9.8.4]`=3, `[9.8.7]`=No.
4. Build the HA automation: watch PV surplus (PV1+PV2 power − house consumption) with hysteresis, publish to `espaltherma/sg/set` (2 = Recommended On, 0 = Free running).
5. Decide on relay module (in progress — evaluating a MakerShop.ie 2-relay module for trigger-voltage/polarity compatibility with `SG_RELAY_HIGH_TRIGGER`).
6. Fix or remove the dead `Room Setpoint` template reference.
7. Solarman: disable the open WiFi AP and change default logger web credentials via the SolarmanPV app.
8. Zappi EV charger integration — still not started.
