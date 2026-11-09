# Quickstart & Validation: Lamp Settings Adjustable from Home Assistant

**Feature**: [spec.md](spec.md) | **Plan**: [plan.md](plan.md) | **Date**: 2026-11-09

This is the constitution's **hardware gate** for feature 002 and holds its **traceability
table** (section 6). It reuses the setup, build, upload and logging steps of
[001 quickstart](../001-pir-ledstrip-fade/quickstart.md) sections 1–5; only what differs is
written here. Every scenario must pass and the results table must be filled in before the
feature counts as delivered. Exception: W16 (summer/winter time), see the plan's Complexity
Tracking.

All commands run from the repository root in PowerShell. The lamp's address is
`kantoorlamp.local` (`device_name: kantoorlamp`); 001's commands that say `buitenlamp.local`
need this address instead.

## 1. Before you start

1. Do 001 quickstart sections 1–5 once (ESPHome installed, secrets in place, lamp added to Home
   Assistant). Set `log_level` to `DEBUG` for the validation runs.
2. Check `timezone` in the `substitutions:` block (default `Europe/Amsterdam`). It must be the
   same zone as Home Assistant's (Settings → System → General), because the lamp uses Home
   Assistant's zone while it is connected (research S6).
3. Compile gate and resource gate, as in 001 section 4:

   ```powershell
   .venv\Scripts\esphome config esphome/buitenlamp.yaml
   .venv\Scripts\esphome compile esphome/buitenlamp.yaml
   ```

   **Expected**: valid configuration; no compiler warnings pointing at
   `esphome/buitenlamp.yaml`; RAM ≤ 75 % and flash ≤ 90 %, written into section 5.
   **Range check**: run `.venv\Scripts\esphome -s fade_out_s 61 compile
   esphome/buitenlamp.yaml`. `-s` overrides the value for this build only, so the file is not
   changed. The compile must stop with the `static_assert` message for `fade_out_s`.
4. Upload over WiFi (001 section 5, with `--device kantoorlamp.local`) and watch the log with
   `--no-states`.

**Where the settings are**: Home Assistant → Settings → Devices & services → ESPHome → the
lamp's device page, section **Configuration**. The scenarios below say "set X to Y" for a change
there. For out-of-range tests use Developer tools → Actions (`number.set_value`), because the
card itself does not let you enter such values.

**Power-cut trials**: cut the power **2 to 5 seconds** after the last change (SC-004). The lamp
stores a change within 2 s (FR-004, research S8). At `DEBUG`, the log line
`[preferences] Writing N items: … written` shows the moment a change reaches flash. One such
line about 60 s after every boot is ESPHome's safe-mode boot counter, not a setting (research
S8).

## 2. Scenarios: adjustable settings

| # | Scenario | Steps | Expected | Spec |
|---|----------|-------|----------|------|
| W1 | Entities and defaults | Open the device page after the first upload of 1.2.0 | Configuration shows On period 60 s, Fade-in time 1.0 s, Fade-out time 5.0 s, Time window start 00:00, Time window end 00:00, Reaction mode `only when dark`; each with its unit. Status shows In time window = on. Record the entity ids in section 5 | FR-001, FR-005, FR-017 |
| W2 | Out of range refused | Developer tools → Actions: `number.set_value` On period to 0.5 and to 90000; Fade-in time to 20; Fade-out time to 0. Then `select.select_option` Reaction mode to `never` | Each number is refused: a warning (in Home Assistant or the lamp's log), the value on the card unchanged, the lamp keeps working with the old value. The unknown option is refused by Home Assistant; Reaction mode is unchanged | FR-002 |
| W3 | On period | In the dark: set On period to 10 s, wave once, stay out of view. Repeat with 2 min and 10 min | "on period ended" 10–11 s, 120–121 s and 600–601 s after `motion OFF`; then the normal fade-out. The card shows each new value within 2 s of entering it | FR-003, FR-007, SC-002, SC-008 |
| W4 | Fade times | In the dark: set Fade-in time to 0.5, 3 and 10 s and Fade-out time to 0.5, 10 and 60 s (pairs 0.5/0.5, 3/10, 10/60); wave once per pair. Then with 3/10: switch on and off from Home Assistant, and change the colour while on | "fade complete" after the counted `motion ON` within ±10 % or ±0.1 s of the fade-in time; fade-out the same against the fade-out time. Home Assistant on/off and the colour change use 3 s and 10 s. An observer next to the strip reports no flicker, and no visible steps apart from the lowest levels at the end of fade-outs longer than 10 s (FR-009, research S10); record what is seen at 60 s | FR-008, FR-009, SC-003 |
| W5 | Change while running | In the dark: On period 2 min; wave; while the light is on, set On period to 10 s. After that cycle, wave again. Then set Fade-out time to 60 s, let the on period end, and set Fade-out time to 1 s halfway through the fade | First on period still 120–121 s; second one 10–11 s. The running 60 s fade-out goes on without a jump and completes about 60 s after it started; the next fade-out takes 1 s | FR-003, edge cases |
| W6 | Kept after power cut and update | With the DEBUG log open: (a) set all six settings to non-default values (for example 90 s, 2.0 s, 8.0 s, 07:30, 22:15, `in time window and dark`); (b) choose a brightness and colour on the Buitenlamp light (for example 30 % red) and switch it off; (c) note the `Writing … written` line after each change; (d) cut the power 2–5 s after the last change, and restore it; (e) from 1 minute after the power returns, leave the lamp alone for 2 min. Repeat (a)–(e) three times, each time with values that differ from the trial before (an unchanged value is not written, so it proves nothing). (f) upload over WiFi with a different default, without editing the file: `.venv\Scripts\esphome -s on_period_s 30 run esphome/buitenlamp.yaml --device kantoorlamp.local --no-states` | (c) Each change reaches flash within 0.5 s (settings) or 1.5 s (brightness and colour). (d) After the power cut the card shows the six chosen values and the chosen brightness and colour, and the lamp uses them: one wave (for the example values: after sunset and before 22:15) keeps the light on for the chosen on period (+1 s) in the chosen colour. (e) No `Writing` line in those 2 min; the single write about 60 s after boot is ESPHome's safe-mode counter (research S8). (f) After the update everything is still as chosen; the new YAML default has no effect | FR-004, FR-005, SC-004 |
| W7 | Home Assistant unreachable | With non-default values: disable the lamp's entry in Home Assistant for 15 min at night; walk past; re-enable | The lamp keeps using the chosen values (log timings); after re-enabling, the card shows them | FR-006 |
| W8 | Ease of change | Time how long it takes to change one setting on the device page, from opening Home Assistant on a phone | Under 1 minute; no PC, cable or upload | SC-001 |

## 3. Scenarios: reaction mode and time window

"In window" means a window that contains the current time, for example start = now − 10 min,
end = now + 10 min. "Outside" means one that does not, for example start = now + 1 h,
end = now + 2 h.

| # | Scenario | Steps | Expected | Spec |
|---|----------|-------|----------|------|
| W9 | Only in time window, daytime | In daylight: Reaction mode `only in time window`; window in → wave; window outside → wave | Inside: the lamp turns on as in 001 V3. Outside: the strip stays off, Motion is still reported, the log says `motion ON, ignored (not allowed)` | FR-011, FR-016, SC-005 |
| W10 | Window ends during motion | Mode `only in time window`, On period 60 s; set the end time 3 minutes from now; trigger the lamp and keep moving in front of the sensor until 2 minutes after the end time | At the end time the log shows `conditions ended, motion no longer counted` and In time window turns off; the light fades out 60–61 s later although you keep moving | FR-012 |
| W11 | Window starts | Mode `only in time window`; set start 2 min from now, end 1 h from now; keep moving in front of the sensor across the start time; then leave and come back | No light while you keep moving across the start; the light comes on at the first new detection after the start | FR-013 |
| W12 | Window across midnight | Mode `only in time window`; start = now + 3 min, end = now − 1 h (crosses midnight). Wave before and after the start time | Before: off, In time window = off. After: the lamp reacts, In time window = on | FR-010, SC-006 |
| W13 | Start equals end | Mode `only in time window`, start = end = 12:00; wave at any time | The lamp reacts; In time window = on | FR-010 |
| W14 | In time window and dark | Mode `in time window and dark`. At night with a window in → wave; at night with a window outside → wave; in daylight with a window in → wave | Reacts only in the first case | FR-011, SC-005 |
| W15 | Always and only when dark | Mode `always`, daylight, window outside → wave. Then mode `only when dark`, night, window outside → wave; daylight, window in → wave | `always`: reacts. `only when dark`: reacts at night whatever the window; does not react in daylight | FR-011, SC-005 |
| W16 | Local time and summer/winter time | (a) Check that In time window changes at the local clock minute in W10–W12. (b) On the next summer/winter time change (28 March 2027), keep a window of 08:00–17:00 and check the In time window history on that day and the next | (a) Changes within 1 minute of the set local time. (b) Changes at 08:00 and 17:00 local time on both days | FR-010, SC-006 |
| W17 | Mode change while on | At night, mode `only when dark`, On period 60 s; trigger the lamp and keep moving; set mode to `only in time window` with a window outside | Log `conditions ended…`; the light fades out 60–61 s later although you keep moving. Setting the mode back makes the next new detection work again | FR-012, edge cases |
| W18 | Time unknown | In daylight, mode `only in time window`, window outside. Switch the WiFi access point off, power-cycle, wave after 30 s; then switch WiFi back on while you keep moving in front of the sensor until the light goes out | With WiFi off the lamp reacts (time unknown). After it reconnects: In time window showed `unknown` before the first time sync, then `off`; daytime motion is ignored again within 2 min. When the time becomes known, the log shows `conditions ended, motion no longer counted`, and the light fades out one on period later although you keep moving (research S4) | FR-012, FR-014 |
| W19 | Manual mode is not limited | Mode `only in time window`, window outside, daylight: switch on from Home Assistant; wave; switch off | On and off work as in 001 V16; Mode = manual while on | FR-015 |
| W20 | Fallback with the reaction mode | (a) Daylight, mode `always`, On period 15 min: switch on from Home Assistant, wave once, disable the lamp's entry. (b) The same with mode `only in time window` and a window outside | (a) 10–11 min after disabling, Mode = automatic and the light stays on, then fades out one on period after the motion ended. (b) 10–11 min after disabling, the light fades out | FR-015 |
| W21 | Window status latency | Mode any; change the window so that it starts to include now, then so that it excludes now. Also time how fast the card confirms each new window time, and change Reaction mode three times and time that too | In time window follows within 2 s each time. The card confirms each new window time and Reaction mode within 2 s | FR-017, SC-008 |
| W22 | Regression with defaults | Reset all six settings to their defaults (W1 values). Run 001 V3, V4, V8, V9, V10 and V11, as often as 001's Trials table asks | Same results as for 1.1.0 | SC-007 |

**Trials**:

| Scenario | Trials | Criterion |
|----------|--------|-----------|
| W3, each value | 5 | SC-002 |
| W4, each pair | 5 | SC-003 |
| W6 (power-cut part) | 3 | SC-004 |
| W9, W14, W15: per mode, inside and outside | 5 each | SC-005 |
| W12 and W16 (a): In time window history on 3 days, one window crossing midnight | 3 days | SC-006 |
| Setting confirmed on the card: 4 checks on On period (W3), 3 on a window time and 3 on Reaction mode (W21) | 10 | SC-008 |
| W21: In time window follows the change | 10 | FR-017 |
| W22: 001 V3 + V4, V8, V11 | 10, 10, 5; V10 over 3 days | SC-007 |

When finished, set `log_level` back to `INFO`, reset any test settings the owner does not want,
and upload over WiFi.

## 4. Failed trials

List each failed trial here with the scenario, the date and what was seen.

## 5. Results (fill in during the hardware gate)

| Item | Value |
|------|-------|
| ESPHome version, framework, variant | ESPHome 2026.10.1, esp-idf, `esp32c3` |
| Flash usage (% and bytes), INFO and DEBUG | INFO 47.5 % (872278 bytes); DEBUG 48.1 % (882638 bytes); limit 90 % |
| Static RAM usage (% and bytes) | INFO and DEBUG 34.4 % (110566 bytes); limit 75 % |
| Entity ids in Home Assistant (W1) | |
| Smoothness at Fade-out time 60 s (W4) | |
| Scenarios W1–W22 passed (list any exceptions) | |
| W16 (b) date and result | |
| Date and `project: version` | 1.2.0 |

## 6. Traceability (constitution gate 3)

Ids are from [data-model.md](data-model.md). In the YAML's comments, this feature's ids are
written `002/FR-0xx`, so they are not mistaken for 001's ids with the same number (plan.md,
IV.3).

| FR | Implemented by | Scenarios |
|----|----------------|-----------|
| FR-001 | `on_period`, `fade_in_time`, `fade_out_time` (template numbers), `window_start`, `window_end` (template time), `reaction_mode` (template select), all `entity_category: config` | W1 |
| FR-002 | number `min_value` / `max_value` (ESPHome refuses out-of-range calls); time and select validation | W2 |
| FR-003 | `on_period_timer` delay and fade lengths read the entities when they start; optimistic entities publish at once | W3, W5 |
| FR-004 | `restore_value: true` on the six entities; `preferences: flash_write_interval: ${flash_write_interval}` (500 ms), which also covers 001's `chosen_*` globals | W6 |
| FR-005 | `initial_value` / `initial_option` from the owner settings; stored value wins | W1, W6 |
| FR-006 | all values stored on the lamp; no dependency on Home Assistant at runtime | W7 |
| FR-007 | `on_period_timer` with `${on_period_now_ms}` | W3, W5 |
| FR-008 | `lamp_fade_on` / `lamp_fade_off` with `${fade_in_now_ms}` / `${fade_out_now_ms}`; `fade_in_time` `on_value` → `set_default_transition_length` | W4 |
| FR-009 | `strip` `gamma_correct`, ESPHome transitions | W4 |
| FR-010 | `in_window` lambda; `sntp_time` and `ha_time` `timezone: ${timezone}` | W12, W13, W16 |
| FR-011 | `motion_allowed` lambda; `motion_started` | W9, W14, W15 |
| FR-012 | `motion_allowed` `on_release` → `conditions_ended` | W10, W17, W18 |
| FR-013 | `motion_started` acts on the rising edge of `motion` only | W11 |
| FR-014 | `motion_allowed` ON while Daylight or In time window has no state | W18 |
| FR-015 | `ha_command` unchanged; `fallback_timer` reads `motion_allowed` | W19, W20 |
| FR-016 | `motion` entity unchanged | W9 |
| FR-017 | `in_window` (template binary sensor, shown in Home Assistant) | W1, W18, W21 |
