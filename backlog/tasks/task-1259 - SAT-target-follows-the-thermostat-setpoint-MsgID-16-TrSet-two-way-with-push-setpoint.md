---
id: TASK-1259
title: >-
  SAT target follows the thermostat setpoint (MsgID 16 TrSet), two-way with
  push-setpoint
status: In Progress
assignee:
  - '@claude'
created_date: '2026-10-19 17:33'
updated_date: '2026-10-20 05:37'
labels:
  - feature
  - sat
milestone: 2.0.0
dependencies: []
priority: medium
ordinal: 391000
---

## Description

<!-- SECTION:DESCRIPTION:BEGIN -->
Feature request by gruijter 2026-10-19, endorsed by Sergeant D. Today SAT only pushes its target to the thermostat (satpushsetpoint, TC= command, SATcontrol.ino ~407-415); a setpoint change on the wall thermostat is ignored, so the thermostat is useless as UI. Request: opt-in 'SAT target follows thermostat setpoint' that takes MsgID 16 TrSet as SAT target when it changes. Must avoid a feedback loop with satpushsetpoint (TC= echo). Sergeant D's rule: thermostat syncs only the room setpoint; with SAT active SAT drives the boiler; on SAT inactive/malfunction fall back to the thermostat. Open: thermostat may send CH enable off when it thinks the room is warm, decide who wins.
<!-- SECTION:DESCRIPTION:END -->

## Acceptance Criteria
<!-- AC:BEGIN -->
- [ ] #1 Design agreed (option name, loop guard, CH-enable arbitration)
- [ ] #2 Thermostat setpoint change is adopted as SAT target within one TrSet cycle, and a SAT target change pushed via TC= does not bounce back (bench or simulator evidence)
- [ ] #3 settings.sat.bFollowThermostat persists as SATfollowthermostat (default false); GET /api/v2/settings exposes satfollowthermostat, POST accepts it, settings.ini still parses as JSON
- [ ] #4 A TrSet never seen (0/NaN) or outside 5-30 is ignored; the first valid value after boot is adopted; a SAT preset set after a thermostat change holds until the next thermostat change; with the switch off nothing changes
- [ ] #5 v2 SAT settings System group shows 'Follow thermostat setpoint' next to Push setpoint; HA discovery publishes the switch and sat/follow_thermostat toggles it
- [x] #6 ./build.sh (esp32-combo) SUCCESS; evaluate.py --quick 0 failures; tests/test_settings_whitelist.py green; docs/api/README.md and MQTT.md updated
- [ ] #7 Bench (requester flashes 192.168.0.32, real T6): thermostat 20 -> target_temp 20 within one control interval; thermostat off -> target follows the frost setpoint and the boiler stays off; switch off -> no following
<!-- AC:END -->

## Implementation Plan

<!-- SECTION:PLAN:BEGIN -->
Design (session 2026-10-20 with the requester on the Classic+S3 Mini bench, mirrors upstream SAT CONF_THERMOSTAT / _async_thermostat_changed, which copies only the target temperature):
1. Setting settings.sat.bFollowThermostat (ini SATfollowthermostat, REST satfollowthermostat, MQTT sat/follow_thermostat -> sat/follow_thermostat_enable, HA switch via streamSatBoolSwitch), default off.
2. Edge-triggered (Sergeant D): once per control tick satFollowThermostatTick() compares OTcurrentSystemState.TrSet (MsgID 16, thermostat T frame, never gateway-substituted) with the last value it acted on. Only a CHANGE that also differs from the current target is applied: settings.sat.fTargetTemp = TrSet (RAM only, like presets), preset cleared, PID integral reset. First observed value counts as a change; TrSet <= 0/NaN (never seen) or outside 5-30 is ignored.
3. Loop guard with satpushsetpoint: edge-triggering is the guard. A TC= push makes the thermostat report the same TrSet, which equals the target -> no-op; a thermostat that ignores the override keeps its old TrSet -> no change -> no-op. No periodic sync, so no ping-pong.
4. CH-enable arbitration: SAT ignores the thermostat's CH bit. The gateway drops the thermostat's original MsgID 0 bit while SAT substitutes it (is_value_valid_for_master_topic, OTGW-Core.ino:1817), and TPI thermostats (Honeywell T6) toggle it as normal cycling. The thermostat's off state arrives as its frost setpoint in TrSet (15 C on the bench T6) and is followed like any value; the ADR-150 cold cutoff then keeps the boiler off. SAT inactive/disabled already releases CS=0 to the thermostat (ADR-179).
5. No 'off threshold' knob (requester's choice: pure follow).
Files: SATtypes.h, settingStuff.ino, restAPI.ino (addBool, knownSettings, /sat alias), MQTTstuff.ino row, MQTTHaDiscovery.cpp switch case, SATcontrol.ino (tick + enable publish), data/v2.js (SAT settings System group next to Push setpoint), docs/api/README.md + MQTT.md. Build combo, evaluate, settings whitelist test; bench by the requester on 192.168.0.32 (they flash).
<!-- SECTION:PLAN:END -->

## Implementation Notes

<!-- SECTION:NOTES:BEGIN -->
2026-10-19 Sergeant D (#alpha-testing): adopt the thermostat setpoint only on a room setpoint CHANGE (edge-triggered), not by periodic sync. CH-enable arbitration question still open.

2026-10-20 maintainer decision: CH enable from the thermostat is ignored while SAT is active and healthy (SAT wins); when SAT is off or faulted the thermostat applies again.

2026-10-19 Sergeant D (#alpha-testing): adopt the thermostat setpoint only on a room setpoint CHANGE (edge-triggered), not by periodic sync. CH-enable arbitration question still open.

2026-10-20: duplicate TASK-1267 was created in this session before finding this task; archived. Requester decisions: pure follow (no off threshold), HA switch for consistency, requester flashes the build themselves.

2026-10-20 implementation (alpha.441, this session): bFollowThermostat + satFollowThermostatTick() in SATcontrol.ino (called at the top of each control tick), SATfollowthermostat ini key, satfollowthermostat REST (GET/POST + /sat alias follow_thermostat), MQTT row sat/follow_thermostat -> SATfollowthermostat, state topic sat/follow_thermostat_enable, HA discovery switch idx 13 (loop bound 13 -> 14 in MQTTstuff.ino), v2 SAT settings System group, docs (api/README.md, api/MQTT.md, c4-code-mqtt.md, c4-code-sat.md, c4-component-smart-thermostat.md). Evidence: ./build.sh esp32-combo firmware SUCCESS 28.3 s + filesystem SUCCESS 2.2 s (alpha.441+80e2ada); strings: satfollowthermostat, SATfollowthermostat, sat/follow_thermostat_enable present in the .ino.bin; evaluate.py --quick exit 0 (72 passed, 0 warnings, 0 failed); tests/test_settings_whitelist.py OK; tests/test_static_integrity.py OK (33). test/host/test_ha_discovery_json.cpp updated for index 13 (loop <= 14, expect s <= 13) but NOT run: the runner needs Visual Studio Build Tools (vswhere.exe), unavailable on this Linux host; run it on the Windows bench. Bench ACs (#2, #3, #4, #5 device part, #7) wait for the requester, who flashes the build themselves; the unit (192.168.0.32) was offline during this session's last checks.

2026-10-20 bench finding (requester, alpha.441, T6): with satpushsetpoint AND satfollowthermostat both on, SAT pushed TC=16, the PIC kept the constant override, the T6 reported TrSet 16 back, and thermostat on/off/setpoint changes never reached SAT (OT log A40091000 TrOverride 16; /api/v2/sat target 16, otmonitor roomsetpoint 16). Telnet 60 s: only CS=10.0 keep-alive and SC= on the wire. Fix folded into this branch: the push block is skipped while bFollowThermostat is on, and satFollowThermostatTick() sends TC=0 once per enable (also at boot) to release any override; nothing released it before when push was switched off. Rebased as feature/sat-follow-thermostat on dev 4b6550197 (alpha.442).
<!-- SECTION:NOTES:END -->
