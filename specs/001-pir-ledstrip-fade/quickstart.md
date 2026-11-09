# Quickstart & Validation: PIR-Controlled LED Strip with Fade (ESPHome)

**Feature**: [spec.md](spec.md) | **Plan**: [plan.md](plan.md) | **Date**: 2026-11-01

This guide shows the feature works from end to end. It is also the constitution's **hardware
gate** and holds the **traceability table** (section 10). Every scenario must pass and the
results table must be filled in before the feature counts as delivered. The one exception is
the 60-day run V22 (see the plan's Complexity Tracking).

All commands run from the repository root in PowerShell.

## 1. Prerequisites

**Hardware**: everything in the bill of materials in
[contracts/hardware.md](contracts/hardware.md), wired as in its pin map. Before applying power,
check two things:
- PIR VCC goes to **3V3**, not 5 V.
- The 74AHCT125 is powered from 5 V.

**PC software**:

| Tool | Version | Notes |
|------|---------|-------|
| Python | 3.13.15 | already installed |
| ESPHome | 2026.10.1 | installed in step 2 into `.venv/` from `requirements.txt` |
| USB driver | built into Windows | the ESP32-C3's USB port appears as a COM port |

**Home Assistant (on the Raspberry Pi)**: nothing to install. The ESPHome integration is built
in. Make sure the Pi and the lamp are on the same network, so mDNS discovery works.

## 2. Install ESPHome (once)

```powershell
python -m venv .venv
.venv\Scripts\python -m pip install -r requirements.txt
.venv\Scripts\esphome version
```

**Expected**: `Version: 2026.10.1`.

## 3. Configure

1. Copy `esphome/secrets.example.yaml` to `esphome/secrets.yaml` and fill in `wifi_ssid`,
   `wifi_password` and `api_encryption_key`. Generate the key with the command in
   [contracts/settings.md](contracts/settings.md).
2. Check that `.gitignore` lists `esphome/secrets.yaml`, `esphome/.esphome/` and `.venv/`
   (FR-022).
3. In the `substitutions:` block of `esphome/buitenlamp.yaml`:
   - Set `latitude` and `longitude` to the installation location.
   - Set `num_leds` to the number of pixels on the strip.
   - For the validation runs, set `log_level` to `DEBUG`.

## 4. Compile gate and resource gate

```powershell
.venv\Scripts\esphome config esphome/buitenlamp.yaml
.venv\Scripts\esphome compile esphome/buitenlamp.yaml
```

The first compile downloads ESP-IDF and takes several minutes.

**Expected**:
- `esphome config` ends with "Configuration is valid!" and prints no warnings for this
  configuration.
- `esphome compile` succeeds with **no compiler warnings that point to
  `esphome/buitenlamp.yaml`**. Lambda warnings are reported against the YAML file and line.
  Warnings from ESPHome or ESP-IDF themselves don't count against the gate.
- The summary shows `RAM: … ≤ 75 %` and `Flash: … ≤ 90 %`. Write both into the results table.
- Range check: temporarily set `num_leds` to `"300"`. The compile must stop with the
  `static_assert` message for `num_leds`. Then set it back.

## 5. First upload (USB), later uploads (WiFi), and the log

First upload, over USB:

```powershell
.venv\Scripts\esphome upload esphome/buitenlamp.yaml --device COM5
```

Replace `COM5` with the board's port (Device Manager → Ports). Every later upload goes over
WiFi:

```powershell
.venv\Scripts\esphome upload esphome/buitenlamp.yaml --device kantoorlamp.local
```

To watch the log, use either of these:

```powershell
.venv\Scripts\esphome logs esphome/buitenlamp.yaml --device kantoorlamp.local --no-states
.venv\Scripts\esphome logs esphome/buitenlamp.yaml --device COM5
```

`--no-states` matters: without it the log viewer counts as a Home Assistant client and holds
off the manual-mode fallback (research R10).

With `log_level: DEBUG`, the scripts log every event with a `millis()` timestamp:
- motion ON or OFF, counted or ignored, and the reason;
- fade start with its computed length, and "fade complete";
- on-period end, mode changes, fallback, time sync, and API client connect or disconnect.

The timing checks below use these timestamps.

## 6. Add the lamp to Home Assistant (V13)

After the first upload, Home Assistant shows a notification: **New device discovered:
Buitenlamp** (Settings → Devices & services). Click **Add**, paste `api_encryption_key`, and
confirm.

## 7. Characterize the sensor (do this once, before the timing scenarios)

The Grove mini PIR's hold time and warm-up time are not published (research R5).

1. Power-cycle the board and watch the raw PIR lines in the log for the first 60 s without
   moving in front of the sensor. Note the last spurious detection; that is the real warm-up
   time. If it is well under 30 s, `pir_stabilization_ms` may be lowered.
2. Wave once, then keep still. The time between `motion ON` and `motion OFF` is the sensor's
   hold time. Record it.

## 8. Validation scenarios

Run the timing scenarios **after dark**, or with the location temporarily moved to where it is
currently night (V9). Times come from the log timestamps unless a scenario says "stopwatch".
Home Assistant checks use the device page (Settings → Devices & services → ESPHome →
Buitenlamp) and its history.

| # | Scenario | Steps | Expected | Spec |
|---|----------|-------|----------|------|
| V1 | Boot is dark and quiet | Power-cycle the board with the strip connected | No flash or random colours on any LED; the strip stays off | FR-009, SC-006 |
| V2 | Stabilization | Wave within the first 30 s after power-up | Nothing happens; Motion stays off; the log shows no counted motion before `pir_ready` | FR-010 |
| V3 | Fade-in | Wave once after stabilization | "fade complete" 0.9–1.1 s after the counted `motion ON`; the fade looks smooth, with no jumps | FR-001, FR-002, FR-007, SC-001, SC-003 |
| V4 | On period and fade-out | Wave once, then stay out of view | On-period end 60–61 s after `motion OFF`; fade-out "fade complete" 4.75–5.25 s later; the strip is fully dark afterwards | FR-003, FR-005, FR-008, SC-002 |
| V5 | Stays on with presence | Move in front of the sensor every 30–45 s for 30 min | No fade-out starts during the test | FR-004, SC-004 |
| V6 | Recovery during fade-out | Let the fade-out start; wave halfway through | Rises from the current level to full within 1 s, never goes dark | FR-006, SC-005 |
| V7 | Smoothness and one colour | Watch 20 full cycles standing next to the strip | No visible flicker, steps or jumps; every LED shows the same colour and brightness; the colour does not shift during a fade | FR-007, FR-016, FR-017, SC-003 |
| V8 | Daylight ignored | In daytime, with the real coordinates: wave | The strip stays off; Daylight = on; the log shows the motion as not counted (daylight) | FR-018, SC-009 |
| V9 | Night via location | In daytime, set `longitude` about 180° away from the real value, upload over WiFi, wave | Daylight = off; the lamp works as in V3. Restore the real value afterwards | Story 3 |
| V10 | Sunset and sunrise | On 3 days, compare Daylight changes in the Home Assistant history with the published sunset and sunrise for the location | Each change within 2 min of the published time | FR-019, FR-021, SC-010 |
| V11 | No WiFi at power-up | In daytime, switch the WiFi access point off, power-cycle, wave after 30 s; then switch WiFi back on | The lamp turns on (time unknown); within 2 min of WiFi returning, daytime motion is ignored again | FR-014, FR-020, FR-023, SC-011 |
| V12 | Sensor unplugged | Disconnect the PIR SIG wire in the dark | The lamp stays off for 5 min; Motion stays off | FR-012 |
| V13 | Discovery | First power-up with Home Assistant running (section 6) | "Discovered: Buitenlamp" within 5 min. After Add and the key: one device with Buitenlamp (light, with brightness and colour), Motion, Daylight, Brightness and Mode; no address typed | FR-024, SC-012 |
| V14 | Status and command latency | Walk past; switch on and off from Home Assistant, and change the brightness or colour (10 times each) | Home Assistant updates within 2 s; each command starts the fade within 1 s | FR-025, FR-027, SC-013 |
| V15 | Offline detection | Unplug the lamp's power | Home Assistant shows the device unavailable within 2 min | FR-026 |
| V16 | Manual on/off | Home Assistant, in daytime: turn on; wave; turn off while standing in view; keep standing; then leave and come back | **On**: fades in within 1 s, stays on, Mode = manual, Motion still reported. **Off**: fades out, Mode = automatic, and the light does **not** come back while you keep standing there; it comes on only at a *new* detection after you leave and return (after dark). Also: "on" while already on from motion gives no visible change but Mode = manual; "off" while off changes nothing | FR-027, SC-013, edge cases |
| V17 | Home Assistant unreachable, lamp keeps working | At night, disable the Buitenlamp entry in Home Assistant for 1 h (device page → ⋮ → Disable); walk past several times; re-enable it | V3/V4 timings still hold; the lamp never restarts (check uptime in the log); after re-enabling, all states are correct within 1 min | FR-014, FR-028, SC-014 |
| V18 | Manual fallback | In daytime: turn on from Home Assistant, then disable the entry. Repeat with re-enabling after 5 min | 10–11 min after disabling, the light fades out and the log shows Mode = automatic. With the 5 min interruption it stays on | FR-029, SC-015 |
| V19 | Daylight unknown | During V11, look at Daylight in Home Assistant once it reconnects, before the first time sync | Daylight showed *unknown* while the time was unknown | FR-023 |
| V20 | Power cut in manual mode | Turn on from Home Assistant, then power-cycle | The lamp starts off and in automatic mode; Home Assistant shows Buitenlamp = off, Mode = automatic | FR-009, edge case |
| V21 | Timing change over WiFi | Set `on_period_ms` to `"120000"`, upload over WiFi, repeat V4. *From 1.2.0: set `on_period_s` to `"120"`; the YAML value only applies while no On period has been chosen in Home Assistant (002)* | On period 120–121 s; nothing else changes; edit plus upload under 5 min with the enclosure closed. Restore the default afterwards | Story 5, FR-013, FR-030, SC-008 |
| V22 | Long run | Leave running for 60 days without a restart, then repeat V3 and V4 | Timings unchanged (started at delivery; see Complexity Tracking) | FR-015, SC-007 |
| V23 | Time from Home Assistant only | In daytime, unplug the router's internet (WAN) cable, keeping the LAN and Home Assistant up; power-cycle the lamp; wave after 30 s | The log shows `time synced (home assistant)` with `t=` under 60000 (within 60 s of power-up) and no `time synced (sntp)`; Daylight = on; the motion is ignored | FR-020 |
| V24 | Update protection and restart | (a) Copy the project, change `api_encryption_key` in the copy's `secrets.yaml`, and try a WiFi upload from the copy while the light is on. (b) Do a normal WiFi upload while the light is on | (a) The upload is refused; the light and timing carry on without interruption. (b) The lamp restarts, comes up off and in automatic mode, and ignores motion for 30 s | FR-030 |
| V25 | Glitch filter | In the dark, with DEBUG log: touch a jumper from 3V3 to the PIR SIG pin as briefly as possible, 10 times; then hold it for 1 s | Brief touches that the log shows as shorter than 50 ms are not reported as motion; the 1 s touch is | FR-011 |
| V26 | Secrets kept apart | Run `git check-ignore -v esphome/secrets.yaml`; search `esphome/buitenlamp.yaml` for `ssid:`, `password:` and `key:` | `.gitignore` is named as the rule; every hit is a `!secret` reference; `git status` never lists `secrets.yaml` | FR-022 |
| V27 | Sunrise during motion | At night, with DEBUG log: find M, the minutes until the published sunrise. Add (M − 5) / 4 to `longitude` (the sun moves 1° every 4 min; above 180, subtract 360), upload over WiFi, and keep moving in front of the sensor from 2 min before until 2 min after the shifted sunrise. Restore the real value afterwards | The log shows `dark period ended, motion no longer counted` when Daylight turns on; the light fades out 60–61 s later although you keep moving | FR-018 |
| V28 | Brightness and colour from Home Assistant (added 2026-11-02) | With DEBUG log. (a) Lamp off: in Home Assistant set brightness 30 % and a colour (for example red). (b) While on, pick another colour. (c) Turn off. (d) After dark (or with V9's longitude), wave once. (e) Power-cycle the lamp, wait 30 s, wave again in the dark. (f) Check the light card after each step | (a) Fades in to 30 % red within about 1 s; Mode = manual. (b) Fades to the new colour in about 1 s, no flash or dip. (c) Fades out, Mode = automatic. (d) Fades in to the chosen brightness and colour; "fade complete" 0.9–1.1 s after `motion ON`; on period and fade-out as in V4 (fade-out 4.75–5.25 s). (e) Starts off and automatic; the wave gives the same chosen brightness and colour. (f) The card shows the chosen brightness and colour | FR-017, FR-027, FR-031, SC-016 |

**Trials**: repeat each scenario as often as its success criterion requires, and list any failed
trial in section 9.

| Scenario | Trials | Criterion |
|----------|--------|-----------|
| V1 and V2, then one wave after stabilization | 5 | SC-006 |
| V3 + V4 (one wave gives both) | 10 | SC-001, SC-002 |
| V6 | 5 | SC-005 |
| V8 | 10 | SC-009 |
| V11 | 5 | SC-011 |
| V18, each variant | 3 | SC-015 |
| V28 | 3 | SC-016 |

V7 (20 cycles) and V14 (10 times) already include their counts.

When finished, set `log_level` back to `INFO` and upload again over WiFi. Delivered firmware
ships at INFO (constitution, Logging).

## 9. Results (fill in during the hardware gate)

| Item | Value |
|------|-------|
| ESPHome version, framework, variant | ESPHome 2026.10.1, esp-idf (ESP-IDF 5.5.6), `esp32c3` |
| Flash usage (% and bytes) | 1.1.0 (2026-11-02, `num_leds` 26): 47.0 % (862 214 of 1 835 008 bytes) at INFO; 47.5 % at DEBUG. (1.0.0, `num_leds` 60: 47.0 %.) Limit 90 % |
| Static RAM usage (% and bytes) | 1.1.0 (2026-11-02, `num_leds` 26): 34.2 % (109 758 of 321 296 bytes) at INFO and DEBUG. (1.0.0, `num_leds` 60: 34.0 %.) Limit 75 % |
| PIR warm-up time measured | |
| PIR hold time measured | |
| Strip current with all four channels at full (A), `num_leds` | |
| Scenarios V1–V28 passed (list any exceptions) | V26 passed 2026-11-01: `secrets.yaml` ignored by `esphome/.gitignore` (ESPHome-generated) and the root `.gitignore`; every `ssid:`, `password:` and `key:` is a `!secret`. Others: |
| V22 start date / end date and result | |
| Date and `project: version` | |

## 10. Traceability (constitution gate 3)

Each functional requirement, the YAML element that implements it (ids from
[data-model.md](data-model.md)), and the scenarios that prove it.

| FR | Implemented by | Scenarios |
|----|----------------|-----------|
| FR-001 | `pir_raw` (gpio binary sensor) → `motion` (template) | V3 |
| FR-002 | `motion` `on_press` → `motion_started` → `lamp_fade_on` | V3 |
| FR-003 | `motion` `on_release` → `motion_ended` → `on_period_timer` | V4, V21 |
| FR-004 | `motion_started` (stops `on_period_timer`), `motion_ended` (restarts it) | V5 |
| FR-005 | `on_period_timer` → `lamp_fade_off` | V4 |
| FR-006 | `motion_started` → `lamp_fade_on` from the current level (`fade_length_ms` from `${strip_level}`) | V6 |
| FR-007 | `strip` `gamma_correct`, ESPHome transitions | V3, V7 |
| FR-008 | `lamp_fade_off` (state 0 at the end) | V4 |
| FR-009 | `strip` `restore_mode: ALWAYS_OFF`; state globals `restore_value: no`; 10 kΩ pull-down | V1, V20 |
| FR-010 | `on_boot` delay → `pir_ready`; `motion` template | V2, V24 |
| FR-011 | `pir_raw` `delayed_on` / `delayed_off` | V25 |
| FR-012 | `pir_raw` pull-down | V12 |
| FR-013 | `substitutions:`; `on_boot` `static_assert` range checks | V21 |
| FR-014 | `wifi`/`api` `reboot_timeout: 0s`; all logic on the device | V11, V17 |
| FR-015 | script `delay`s on the 64-bit scheduler | V22 |
| FR-016 | one light entity drives the whole strip; no effects | V7 |
| FR-017 | `chosen_*` globals (initial values from the `on_*` substitutions) used by `lamp_fade_on` and `lamp_fade_off`; fade lengths against `chosen_brightness` | V7, V28 |
| FR-018 | `motion_started` (Daylight check); `dark_period_ended`; `on_period_timer` | V8, V10, V27 |
| FR-019 | `sun`, `sun_elevation`, `daylight`, `sun_horizon_deg` | V10 |
| FR-020 | `sntp_time`, `ha_time`, time-retry `interval` | V11, V23 |
| FR-021 | darkness from elevation, not local time | V10 |
| FR-022 | `secrets.yaml` (git-ignored), `secrets.example.yaml` | V26 |
| FR-023 | `daylight` has no state → counts as dark (`motion_started`, `fallback_timer`) | V11, V19 |
| FR-024 | `api` encryption, mDNS, entities; `strip` light `name: None` | V13 |
| FR-025 | `strip` (light), `motion`, `daylight`, `brightness`, `mode_text` (published by `on_boot`, `manual_on`, `manual_off`, `fallback_timer`) | V14 |
| FR-026 | native API availability | V15 |
| FR-027 | `strip` `on_state` (when `lamp_call_active` is clear) → `ha_command` → `manual_on` / `manual_off`; `manual_mode`, `need_fresh_motion`; `lamp_fade_off` from the current level; `default_transition_length` for colour changes | V14, V16, V28 |
| FR-028 | API reconnect; `reboot_timeout: 0s` | V17 |
| FR-029 | `api: on_client_disconnected` → `fallback_timer` (checks Home Assistant at expiry; reads `motion_counted`, `on_period_timer`) | V18 |
| FR-030 | `ota` (esphome, encryption); safe mode; restart behaviour | V21, V24 |
| FR-031 | `chosen_*` globals `restore_value: yes` (saved only when changed), set by `ha_command`; `on_boot` → `lamp_fade_off` loads them into the strip | V28 |
