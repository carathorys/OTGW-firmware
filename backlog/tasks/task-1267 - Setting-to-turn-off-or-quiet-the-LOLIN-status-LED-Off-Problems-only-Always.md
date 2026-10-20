---
id: TASK-1267
title: >-
  Setting to turn off or quiet the LOLIN status LED (Off / Problems only /
  Always)
status: In Progress
assignee:
  - '@claude'
created_date: '2026-10-20 05:37'
updated_date: '2026-10-20 05:37'
labels: []
dependencies: []
ordinal: 397000
---

## Description

<!-- SECTION:DESCRIPTION:BEGIN -->
Since alpha.420 (TASK-1221) the LOLIN S3 Mini / S3 Mini Pro on-board WS2812B pixel blinks at 1 Hz as a WiFi status light (green connected, red no WiFi, purple AP up). There is no way to turn it off; a user on an OTGW Classic + S3 Mini finds the constant green blink annoying (2026-10-20). Add a persisted setting with three modes: Off (always dark), Problems only (dark while WiFi STA is connected and no AP is up; red/purple blink otherwise), Always (current behaviour, the default). The ticker callback reads the mode on every tick, so changes apply without a reboot. Setting is gated on HAS_STATUS_PIXEL (combo builds); on an OTGW32 the pixel is not driven, so the setting has no effect there. 2.0.0 (dev) only: the pixel code does not exist on otgw-1.x.x.
<!-- SECTION:DESCRIPTION:END -->

## Acceptance Criteria
<!-- AC:BEGIN -->
- [x] #1 settings.iStatusPixelMode (0=Off, 1=Problems only, 2=Always, default 2) is persisted as StatusPixelMode in settings.ini, read back clamped to 0-2, and settings.ini still parses as JSON (json.load) after a save
- [ ] #2 statusPixelTick() keeps the pixel dark in Off, dark while connected (red/purple blink otherwise) in Problems only, and unchanged in Always; a mode change takes effect without a reboot
- [x] #3 GET /api/v2/settings lists statuspixelmode (int 0-2) on HAS_STATUS_PIXEL builds; POST /api/v2/settings accepts it and the value survives a reboot
- [x] #4 v2 web UI Settings > Behaviour shows a 'LOLIN status LED' select with Off / Problems only / Always
- [x] #5 ./build.sh --target all builds esp32, esp32-classic and esp32-combo (SUCCESS); python evaluate.py --quick reports 0 failures
- [ ] #6 On a Classic + LOLIN S3 Mini (192.168.0.32): the old firmware has no statuspixelmode; the new firmware shows it, and the user confirms the LED behaviour in each of the three modes
<!-- AC:END -->

## Implementation Plan

<!-- SECTION:PLAN:BEGIN -->
User-approved 2026-10-20 (three modes, default Always, npx backlog.md, Claude flashes over WiFi via safeboot, commit locally, no push).
1. OTGW-firmware.h: uint8_t iStatusPixelMode = 2 in OTGWSettings next to bLEDblink.
2. settingStuff.ino: writeJsonIntKV StatusPixelMode; updateSetting clamps 0-2 (default 2); debug dump line.
3. OTGW-firmware.ino statusPixelTick(): Off -> black; Problems only -> black while STA connected and no AP; Always -> unchanged.
4. restAPI.ino: addInt statuspixelmode 0-2 and knownSettings entry, both under HAS_STATUS_PIXEL.
5. data/v2.js: behavior label 'LOLIN status LED' + ENUM_OPTS Off / Problems only / Always.
6. handleDebug.ino + docs/api/README.md: show/document the setting.
7. bin/bump-prerelease.sh, ./build.sh --target all, evaluate.py --quick, settings.ini json.load check.
8. Bench on 192.168.0.32 (Classic + S3 Mini): GET settings before (absent) vs after (present), app-only OTA via /safeboot (settings kept), set each mode via POST, user confirms LED, reboot, value persists.
<!-- SECTION:PLAN:END -->

## Implementation Notes

<!-- SECTION:NOTES:BEGIN -->
2026-10-20 evidence (unit 192.168.0.32, Classic + LOLIN S3 Mini, status_pixel_gpio 47):
- Old firmware alpha.436+22aab83: GET /api/v2/settings has no statuspixelmode; POST statuspixelmode -> HTTP 400 {"error":"Unknown setting name"}.
- ./build.sh --target all: esp32, esp32-classic, esp32-combo all SUCCESS (alpha.440+828bad6). strings: statuspixelmode in combo+classic .ino.bin, absent from otgw32 (HAS_STATUS_PIXEL gate); StatusPixelMode key in all three.
- python3 evaluate.py --quick: exit 0, 72 passed, 0 warnings, 0 failed. tests/test_static_integrity.py 33 OK, tests/test_settings_whitelist.py 3 OK, tests/test_webui_asset_versioning.py 2 OK, node tests/webui/settings-error-toast 4/4, v2-settings-sat-followups 26/26.
- WiFi update: POST /safeboot 200 -> safeboot 1.0.0 up in ~6 s; POST /update (field firmware, combo .ino.bin sha256 9476ac46...) 200 in 9.7 s -> alpha.440 up in ~11 s, PIC mode, MQTT + OT connected. Six changed web files uploaded singly via /upload (303 each) and read back byte-identical (v2.js, v2-bundle.css, v2.html, fsexplorer.css, FSexplorer.html, version.hash); state files (sat_pid_state, energy, dhw_water, ot-boiler/thermo) unchanged.
- Settings before/after: 176 -> 177 keys, only statuspixelmode added. sattargettemp 15 -> 20 is the restart dropping a RAM-only SAT preset (Away 15), not this change.
- POST statuspixelmode 0 -> 200, GET value 0; settings.ini has "StatusPixelMode": 0 and json.load OK (212 keys). POST 7 -> stored 2 (default, same rule as boardmode); set back to 0.
- Headless Chromium on the unit: Settings > Behavior shows 'LOLIN status LED' select [0=Off, 1=Problems only, 2=Always], value 0, 0 page exceptions, 0 non-GET requests (screenshot checked).
- GET /ReBoot: bootcount 6 -> 7, still alpha.440, statuspixelmode still 0.
- Pending: user visual confirmation of the LED in each mode (AC#2 visual part, AC#6).
<!-- SECTION:NOTES:END -->
