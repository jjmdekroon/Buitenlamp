# Implementation Plan: PIR-Controlled LED Strip with Fade

**Branch**: `001-pir-ledstrip-fade` | **Date**: 2026-11-01, updated 2026-11-02 (brightness and
colour from Home Assistant) | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/001-pir-ledstrip-fade/spec.md`

**Note**: This template is filled in by the `/speckit-plan` command; its definition describes the execution workflow.

## Summary

An ESP32-C3 running **ESPHome** drives an SK6812 RGBW LED strip from a Seeed Grove mini PIR
sensor. This plan replaces the earlier hand-written Arduino design (constitution v2.0.0).

- **Fading**: the strip fades on in 1 s, stays on until 60 s after the last motion, then fades
  off over 5 s.
  - ESPHome's built-in transitions do the fading, with gamma correction so the fades look even.
  - Fades that start from a partial level run at the normal rate.
- **Darkness**:
  - Motion only counts between sunset and sunrise. ESPHome's `sun` component works this out
    from the configured location.
  - The time comes from an internet time server **and** from Home Assistant, whichever answers
    first.
  - While the time is unknown, the lamp reacts to motion day and night.
- **Home Assistant**:
  - The lamp connects through the built-in ESPHome integration, over the encrypted native API.
  - Home Assistant discovers it; the owner clicks Add and pastes the key.
  - It shows a light "Buitenlamp" (on/off, brightness and colour), Motion, Daylight, Brightness
    and Mode. The light is the strip itself, exposed to Home Assistant.
  - "On" from Home Assistant, including choosing a brightness or colour, means manual mode
    until "off". After 10 minutes without Home Assistant, the lamp falls back to automatic.
  - The chosen brightness and colour become the on-setting for every "on", from motion too.
    They are stored in flash when they change, so they survive a power cut (FR-031).
- **Updates** go over WiFi, protected by the same encryption key.

**How it is built**: one YAML file with every setting in a `substitutions:` block at the top.
- The lamp logic is a handful of named scripts, each labelled with the requirements it
  implements.
- Timers are script delays, not `millis()` arithmetic.
- Validation is `esphome config` and `esphome compile` plus the on-device scenarios in
  [quickstart.md](quickstart.md). There are no host-side tests.

Decisions and alternatives are in [research.md](research.md).

## Technical Context

**Language/Version**: ESPHome YAML with short C++ lambdas, as compiled by ESPHome 2026.10.1's
esp-idf toolchain. Lambdas use no exceptions and no RTTI.

**Primary Dependencies** (exact versions):
- **ESPHome 2026.10.1**, pinned in `requirements.txt` and installed in `.venv/` with Python
  3.13.15. The YAML also sets `min_version: 2026.10.1`.
- The **esp-idf** framework, as selected and downloaded by ESPHome 2026.10.1.
- ESPHome components used (no external components):
  - `esp32`, `wifi`, `api`, `ota` (`esphome`), `logger`;
  - `time` (`sntp`, `homeassistant`), `sun`;
  - `light` (`esp32_rmt_led_strip`), `binary_sensor` (`gpio`, `template`), `sensor` (`sun`,
    `template`), `text_sensor` (`template`). The `switch` (`template`) of version 1.0.0 was
    replaced by the light on 2026-11-02;
  - `globals`, `script`, `interval`.
- Home Assistant's built-in ESPHome integration. No add-on is needed on the Pi.

**Storage**: none for lamp state. The light uses `ALWAYS_OFF` and every state global uses
`restore_value: no`, so every boot starts off and automatic (FR-009, FR-030). The only stored
values are the six `chosen_*` globals (the on-setting: brightness, colour brightness, red,
green, blue, white) with `restore_value: yes`; ESPHome writes them to flash only when they
change (FR-031, research R18). ESPHome also stores its own safe-mode boot counter.

**Testing**: no host-side tests (constitution v2.0.0).
- The compile gate is `esphome config` and `esphome compile`, with `static_assert` range checks
  on the settings.
- The hardware gate is quickstart scenarios V1–V28.
- The traceability gate is the FR table in quickstart section 10.

**Target Platform**:
- ESP32-C3 (`esp32: variant: esp32c3`): single-core RISC-V, 160 MHz, 400 KB SRAM, 4 MB flash,
  no PSRAM, framework esp-idf.
- Home Assistant on a Raspberry Pi in the same LAN.

**Project Type**: ESPHome device configuration (embedded firmware).

**Performance Goals**:
- Fade timing within ±0.1 s (fade-in) and ±0.25 s (fade-out); on period +1/−0 s.
- Strip refresh at most 100 Hz.
- Status in Home Assistant within 2 s; commands act within 1 s.
- Offline shown within 2 min.
- Darkness changes within 2 min of the published sunset and sunrise.

**Constraints**:
- No lambda runs longer than 10 ms and no lambda blocks.
- `num_leds` ≤ 240.
- No allocation in lambdas that run repeatedly.
- Static RAM ≤ 75 % and flash ≤ 90 %.
- Must work fully without WiFi, internet or Home Assistant: `wifi` and `api` have
  `reboot_timeout: 0s`.
- PIR powered from 3.3 V; strip data level-shifted to 5 V.

**Scale/Scope**: one device, one strip of up to 240 pixels, one Home Assistant instance. About
250–300 lines of YAML.

**Unknowns**: none left. Each one was settled in research or with the owner:

| Earlier question | Where resolved |
|------------------|----------------|
| ESPHome version and framework | R1 |
| Board and pins | R2 |
| Strip driver keys in 2026.10 | R3 |
| Level shifting | R4 |
| PIR behaviour (unpublished values) | R5; measured in quickstart section 7 |
| Fade curve and partial fades | R6 |
| Darkness | R7 |
| Two time sources | R8, plus a spec clarification |
| Switch instead of light | R9, plus a spec clarification |
| Offline detection time | R9; spec changed to 2 min |
| Home Assistant reachability | R10 |
| Logic structure | R11 |
| Updates and safe mode | R12 |
| Watchdog and reboots | R13 |
| Validation | R14 |
| Memory | R15 |
| Security | R16 |
| Logging | R17 |
| Brightness and colour from Home Assistant; telling its commands apart from motion | R18 (2026-11-02) |

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

The checks below are against constitution v2.0.0.

| # | Rule | Pre-design | Post-design: how the design meets it |
|---|------|------------|--------------------------------------|
| I.1 | Outputs safe from boot | PASS | `strip`: `restore_mode: ALWAYS_OFF`; 10 kΩ pull-down on the data line ([hardware.md](contracts/hardware.md)) |
| I.2 | Defined state after reset; no persisted lamp state | PASS* | On/off, mode and timers never persist (data-model). Only the on-setting (`chosen_*`) persists, because FR-031 asks for it. *Safe mode exception under Complexity Tracking* |
| I.3 | Task watchdog enabled | PASS | `esp32` default `watchdog_timeout: 5s`; never changed or disabled (R13) |
| I.4 | High-current loads through a driver | PASS | Strip on its own 5 V supply; GPIO5 only drives data through the 74AHCT125 |
| I.5 | Switching inputs filtered | PASS | `pir_raw`: `delayed_on` and `delayed_off` of 50 ms plus 30 s stabilization; darkness is computed, so it cannot chatter |
| II.1 | Lambdas never block; waits use delay/wait_until/scripts | PASS | All waiting in scripts (`on_period_timer`, `fallback_timer`, fade logs, `on_boot`); lambdas only compute values |
| II.2 | Time arithmetic survives rollover | PASS | No `millis()` comparisons at all; the scheduler is 64-bit (R11) |
| II.3 | More than two modes → explicit state machine | PASS* | States AutoOff, AutoOn and ManualOn come from `manual_mode` plus the strip's target ([data-model.md](data-model.md)). T1, T3–T5, T7, T9, T10 and T13 are named scripts; the light finishes T2, T6 and T8 itself; T11 is the `api:` trigger and T12 is `on_boot`. *Deviation under Complexity Tracking* |
| II.4 | No custom ISRs | PASS | None; the PIR is read through ESPHome's gpio binary sensor |
| III.1 | No allocation in repeated lambdas | PASS | Repeated lambdas return `bool` or `float` only; Mode text is published only on change |
| III.2 | RAM ≤ 75 %, flash ≤ 90 % | PASS (to verify) | Expected flash 50–60 %, RAM 15–25 % (R15); measured in quickstart section 4 |
| III.3 | Flash writes only for spec-required values | PASS | `restore_value: yes` only on the `chosen_*` globals (FR-031), written only when they change; light `ALWAYS_OFF`. The light's own `RESTORE_AND_OFF` was rejected because it saves the on/off state at every motion cycle (R18) |
| IV.1 | Hardware declared once; automations use ids | PASS | Pins only in `pir_raw` and `strip`, via substitutions; everything else uses ids |
| IV.2 | Logic in named scripts; short lambdas; no duplication | PASS | Ten scripts (R11, R18); "motion counts" decided only in `motion_started`; light calls only in the two fade scripts; Home Assistant commands handled only in `ha_command` |
| IV.3 | Scripts labelled with FR ids | PASS | Required by the tasks; checked at the review gate |
| IV.4 | Every rule covered by a quickstart scenario | PASS | Quickstart section 10: FR-001 to FR-031 each map to at least one of V1–V28 |
| V.1 | Tunables in one `substitutions:` block | PASS | [settings.md](contracts/settings.md): owner settings plus a labelled block of fixed constants |
| V.2 | Secrets only in a git-ignored `secrets.yaml` | PASS | `wifi_ssid`, `wifi_password`, `api_encryption_key`; `secrets.example.yaml` committed |
| V.3 | Pin map matches the YAML | PASS | [hardware.md](contracts/hardware.md) refers to `pin_pir` and `pin_led_data` |
| V.4 | ESPHome version pinned; external components justified | PASS | `esphome==2026.10.1` and `min_version`; no external components |
| V.5 | YAGNI | PASS | No `web_server`, `captive_portal`, WiFi AP, strip refresh timer or effects. Brightness and colour control is now required (FR-027) and uses the strip light itself, with no extra entity |
| P.1 | Board and framework named | PASS | ESP32-C3, `variant: esp32c3`, esp-idf (R1, R2) |
| P.2 | Build with the ESPHome CLI; zero lambda warnings | PASS | Quickstart section 4 |
| P.3 | Layout: device YAML, secrets, optional packages, no test folder | PASS | See Project Structure |
| P.4 | Home Assistant over the encrypted native API | PASS | `api: encryption: key: !secret api_encryption_key` |
| P.5 | Works without network or Home Assistant; reboot timeouts disabled | PASS | `wifi` and `api` `reboot_timeout: 0s`; all logic local; V11, V17 |
| P.6 | One logger level; INFO when delivered | PASS | `logger: level: ${log_level}`, default INFO |
| G | Quality gates 1–5 | PASS | Compile and resource: quickstart section 4. Traceability: section 10. Hardware: sections 6–8. Review: this table |

**Gate result**: PASS. Three items are justified under Complexity Tracking: the safe-mode
behaviour (I.2), the lamp state held partly in the light component (II.3), and V22 finishing
after delivery (hardware gate).

## Project Structure

### Documentation (this feature)

```text
specs/001-pir-ledstrip-fade/
├── plan.md              # This file (/speckit-plan command output)
├── research.md          # Phase 0 output (/speckit-plan command)
├── data-model.md        # Phase 1 output (/speckit-plan command)
├── quickstart.md        # Phase 1 output (/speckit-plan command)
├── contracts/           # Phase 1 output (/speckit-plan command)
│   ├── ha-api.md        #   Home Assistant entities, commands, availability (native API)
│   ├── settings.md      #   substitutions and secrets: defaults, ranges, units
│   └── hardware.md      #   pin map, wiring, power, bill of materials
├── checklists/
│   └── requirements.md  # spec quality checklist (/speckit-specify)
└── tasks.md             # Phase 2 output (/speckit-tasks command - NOT created by /speckit-plan)
```

### Source Code (repository root)

```text
.gitignore                    # ignores esphome/secrets.yaml, esphome/.esphome/ and .venv/
requirements.txt              # esphome==2026.10.1, the pinned build tool
esphome/
├── buitenlamp.yaml           # the whole device, in this order:
│                             #   1. substitutions: owner settings, then fixed constants
│                             #   2. esphome (incl. on_boot) / esp32 / logger / wifi / api / ota
│                             #   3. time / sun                                   (clock and darkness)
│                             #   4. light                                        (the strip)
│                             #   5. globals / script / interval                  (lamp logic)
│                             #   6. binary_sensor / sensor / text_sensor          (inputs, HA entities)
│                             # ESPHome only accepts on_boot inside esphome:, so the boot logic sits
│                             # in section 2. ESPHome rejects a top-level key that appears twice, so
│                             # binary_sensor: and sensor: each appear once, in section 6: internal
│                             # inputs (pir_raw, sun_elevation) first, then the HA entities.
├── .gitignore                # generated by ESPHome: /.esphome/ and /secrets.yaml
├── secrets.example.yaml      # template; the owner copies it to secrets.yaml
└── secrets.yaml              # git-ignored: wifi_ssid, wifi_password, api_encryption_key
```

**Structure Decision**: this is a single ESPHome device.
- One YAML file, about 250–300 lines, is easier to read and review than a split into
  `packages/` (YAGNI). The constitution allows packages if the file ever grows.
- The section order keeps hardware declarations apart from logic and from the Home Assistant
  entities (Principle IV).
- The earlier `firmware/` sketch layout and the `test/` folder are not created.

### Runtime structure

- Everything project-related runs in ESPHome's single main loop: components, scripts and the
  API.
- WiFi and lwIP run in their own ESP-IDF tasks, managed by the framework.
- The RMT peripheral sends LED data in the background.
- No project code creates tasks or interrupts.

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| I.2: in ESPHome **safe mode** the light component is not started, so after a crash loop a strip that was lit keeps its last frame until a power cycle (R12) | Safe mode (10 failed boots in a row) is what lets a broken update be replaced over WiFi, so the enclosure never has to be opened (FR-030) | Disabling `safe_mode` makes a crashing update recoverable only over USB. Safe mode only follows a crash loop, and every normal boot, reset and update starts the strip off |
| II.3: the lamp state is not one named value in a global or `select`. The mode is the `manual_mode` global; the on/off target and the running fade belong to the `strip` light component | The light already holds the target and the fade. A state global would be a second copy that every transition has to update, including the ones the light finishes on its own (T2, T6, T8), and the two copies could disagree in the middle of a fade | A `select` or enum global for AutoOff/AutoOn/ManualOn adds an entity and string comparisons without making the logic clearer (R11). The three states and all transitions are named and listed in [data-model.md](data-model.md), and every script carries its FR ids |
| Hardware gate: scenario **V22 (60 days)** finishes after delivery, not before | FR-015 / SC-007 need 60 days of uptime to observe directly. The host rollover tests that used to cover this were dropped by constitution v2.0.0 | Holding delivery for 60 days gains nothing: no project code does `millis()` arithmetic, and ESPHome's scheduler keeps time in 64 bits (R11). V22 starts at delivery and its result is recorded in the quickstart |
