# Implementation Plan: Lamp Settings Adjustable from Home Assistant

**Branch**: `002-ha-tunable-settings` | **Date**: 2026-11-09 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/002-ha-tunable-settings/spec.md`

**Note**: This template is filled in by the `/speckit-plan` command; its definition describes the execution workflow.

## Summary

The owner can change six settings of the lamp on its Home Assistant device page instead of
editing the YAML and uploading it again:
- the **on period**, **fade-in time** and **fade-out time**, as number entities in seconds;
- a daily **time window** (start and end), as time entities;
- a **reaction mode** that decides when motion counts: `only when dark` (as before),
  `only in time window`, `in time window and dark`, or `always`.

The values are stored on the lamp within 2 seconds of a change, so they survive a power cut
right after the change, and every update. Until the owner changes them, the defaults from the
YAML apply, so 1.2.0 behaves exactly like 1.1.0 out of the box.

**Approach** (all built-in ESPHome components, no external components):
- `number`, `datetime` (`type: time`) and `select` entities on the `template` platform, with
  `optimistic` and `restore_value`, in the Configuration section of the device page.
- One new internal decision point, `motion_allowed`, holds the reaction-mode rule. The scripts
  that used to check Daylight (`motion_started`, `fallback_timer`) now read it, and its
  `on_release` runs `conditions_ended` (001's `dark_period_ended`, renamed), so every way the
  conditions can end behaves like sunset-to-sunrise did.
- A new status entity, **In time window**.
- The scripts read the entity values when a fade or on period starts, so a running one is never
  changed.
- ESPHome's flash write interval goes from 60 s to 500 ms. Every change is then in flash within
  2 s: the six settings within 0.5 s, and 001's brightness and colour within 1.5 s. Flash is
  still written only for values that changed.

Decisions and alternatives are in [research.md](research.md) (S1–S11).

## Technical Context

**Language/Version**: ESPHome YAML with short C++ lambdas, compiled by ESPHome 2026.10.1's
esp-idf toolchain; unchanged from 001.

**Primary Dependencies**:
- ESPHome 2026.10.1 (pinned in `requirements.txt`, `min_version` in the YAML), esp-idf.
- Components added by this feature: `number` (`template`), `select` (`template`), `datetime`
  (`template`, `type: time`). One more `binary_sensor` (`template`) is shown in Home Assistant
  and one is internal. The built-in `preferences` component gets an explicit
  `flash_write_interval`. No external components.
- Home Assistant's built-in ESPHome integration; Home Assistant 2026.3 or newer sends its time
  zone to the lamp (research S6), older versions work with the zone from the YAML. Both zones
  must be the same.

**Storage**: six more stored values, one per new setting entity (`restore_value: true`).
`flash_write_interval` 500 ms stores every change within 2 s, for these six and for 001's six
`chosen_*` brightness and colour values (research S8). ESPHome writes a value only when it
differs from what is in flash. Lamp state is still never stored.

**Testing**: no host-side tests (constitution v2.0.0). Compile gate with `static_assert` range
checks on the defaults; hardware gate W1–W22 in [quickstart.md](quickstart.md) plus the 001
regression scenarios listed in W22; traceability in quickstart section 6.

**Target Platform**: ESP32-C3 (`variant: esp32c3`, esp-idf), unchanged; Home Assistant on the
Raspberry Pi.

**Project Type**: ESPHome device configuration (embedded firmware).

**Performance Goals**:
- A setting is confirmed in Home Assistant within 2 s.
- A change is in flash within 2 s: ≤ 0.5 s for the six settings, ≤ 1.5 s for brightness and
  colour.
- In time window follows a change within 2 s, and a window edge within 1 min.
- The timing tolerances of FR-007 and FR-008.

**Constraints**: as 001 (no blocking lambda, no allocation in repeated lambdas, RAM ≤ 75 %,
flash ≤ 90 %, works without network). The two template binary sensors evaluate their lambdas
every loop pass; both only read stored values and the clock. The preferences sync runs every
500 ms, but only touches flash when a value is queued.

**Scale/Scope**: one device; seven new entities; about 80–120 lines of YAML changed or added
in `esphome/buitenlamp.yaml`.

**Unknowns**: none left.

| Question | Where resolved |
|----------|----------------|
| How to expose editable, stored values in Home Assistant | S1 (numbers), S2 (time), S3 (select) |
| Units of the owner settings | S1: seconds, renamed |
| A value between steps, sent by an automation | S1: accepted as it is |
| Where the reaction-mode rule lives | S4 (`motion_allowed`) |
| How "In time window" is computed and how fast it updates | S2, S5 |
| Which time zone the window uses, and on which time source it is set | S6: both sources |
| When a changed value takes effect; Home Assistant's own fades | S7 |
| How fast a change reaches flash; flash wear; power cut during a write | S8: 500 ms interval |
| Defaults versus stored values | S9 |
| Smoothness at 0.1 s and 60 s | S10: steps at the lowest levels accepted (FR-009); checked in W4 |
| Memory and compatibility | S11 |

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

Against constitution v2.0.0. Only rules this feature touches are argued; the rest stay as in
the [001 plan](../001-pir-ledstrip-fade/plan.md) and are re-checked at the review gate.

| # | Rule | Pre-design | Post-design: how the design meets it |
|---|------|------------|--------------------------------------|
| I.1 | Outputs safe from boot | PASS | Unchanged: `strip` `ALWAYS_OFF` |
| I.2 | Defined state after reset; no persisted lamp state | PASS | Stored values are settings, not lamp state: on/off, mode and timers still start from their initial values (FR-004 asks for the settings to persist) |
| I.5 | Switching inputs filtered | PASS | The window is computed from the clock with minute resolution and changes at most twice a day; Daylight unchanged; PIR filters unchanged |
| II.1 | Lambdas never block | PASS | New lambdas (`in_window`, `motion_allowed`, ms conversions) only compute values |
| II.2 | Time arithmetic survives rollover | PASS | No `millis()` arithmetic; the window uses wall-clock hour and minute; timers stay script delays |
| II.3 | More than two modes → explicit state machine | PASS* | The reaction mode is a `select` (named options); lamp states unchanged. *001's Complexity Tracking entry for II.3 still applies* |
| III.1 | No allocation in repeated lambdas | PASS | `motion_allowed` reads `active_index()`, not the option text; `in_window` uses `ESPTime` (a struct) |
| III.2 | RAM ≤ 75 %, flash ≤ 90 % | PASS (to verify) | Estimate +15–30 KB flash (S11); measured in quickstart section 1 |
| III.3 | Flash writes only for spec-required values, only on change | PASS | The six entities (FR-004) and 001's brightness and colour (001/FR-031), nothing else. ESPHome writes a key only when its content differs from flash, at most once per value per 500 ms; brightness and colour are checked once a second. The 448 KB NVS area spreads the writes, so wear stays negligible (S8) |
| IV.1 | Hardware declared once | PASS | No new hardware |
| IV.2 | Logic in named scripts; short lambdas; no duplication | PASS | The reaction rule exists once (`motion_allowed`); the window rule once (`in_window`); each seconds→ms conversion once (shared substitution); "motion counts" still only in `motion_started` |
| IV.3 | Scripts labelled with FR ids | PASS | Required by the tasks; changed scripts get this feature's ids next to 001's. In the YAML, 001's ids keep their plain form (`FR-012`) and this feature's ids carry the prefix `002/` (`002/FR-012`), because both specs use the same numbers |
| IV.4 | Every rule covered by a quickstart scenario | PASS | Quickstart section 6: FR-001 to FR-017 each map to at least one of W1–W22 |
| V.1 | Tunables in one `substitutions:` block | PASS* | All defaults, ranges, steps, option indexes and the time zone are substitutions. The live values are Home Assistant entities, which the spec requires; *recorded under Complexity Tracking* |
| V.4 | ESPHome version pinned; no unjustified external components | PASS | Unchanged pin; built-in components only |
| V.5 | YAGNI | PASS | Only what the spec asks: six settings and the In time window status. `motion_allowed` stays internal; no reset button, no per-weekday windows |
| P.5 | Works without network or Home Assistant | PASS | Values are stored on the lamp; without the time, motion counts in every mode (FR-014); without Home Assistant the window uses `timezone` (S6) |
| P.6 | One logger level; INFO when delivered | PASS | Unchanged |
| G | Quality gates 1–5 | PASS | Compile and resource: quickstart section 1. Traceability: section 6. Hardware: sections 2–3. Review: this table |

**Gate result**: PASS, with two items under Complexity Tracking.

## Project Structure

### Documentation (this feature)

```text
specs/002-ha-tunable-settings/
├── plan.md              # This file (/speckit-plan command output)
├── research.md          # Phase 0 output: S1–S11
├── data-model.md        # Phase 1 output: delta on 001's data model
├── quickstart.md        # Phase 1 output: W1–W22, results, traceability
├── contracts/
│   ├── ha-api.md        #   new Home Assistant entities and commands (1.2.0)
│   └── settings.md      #   changed and new substitutions
├── checklists/
│   └── requirements.md  # spec quality checklist (/speckit-specify)
└── tasks.md             # Phase 2 output (/speckit-tasks command - NOT created by /speckit-plan)
```

### Source Code (repository root)

```text
esphome/
└── buitenlamp.yaml      # changed; section order stays as in 001:
                         #   1. substitutions: renamed timing defaults (s), new defaults,
                         #      timezone, range constants, mode indexes, ms conversions,
                         #      flash_write_interval
                         #   2. esphome: project version 1.2.0; on_boot static_asserts on the
                         #      new names; new preferences: block with flash_write_interval
                         #   3. time: sntp_time and ha_time both get timezone: ${timezone}
                         #   4. light: unchanged except the default transition comment/value
                         #   5. scripts: motion_started, conditions_ended (renamed),
                         #      on_period_timer, lamp_fade_on, lamp_fade_off, fallback_timer
                         #   6. entities: new number, select and datetime blocks; in_window and
                         #      motion_allowed in binary_sensor; daylight loses on_press
README.md                # settings section: the Home Assistant settings and the renamed defaults;
                         # kantoorlamp.local instead of buitenlamp.local in the commands
specs/001-pir-ledstrip-fade/contracts/settings.md   # pointer to the 002 changes
specs/001-pir-ledstrip-fade/research.md             # R18: pointer to 002 S8 (2 s instead of 60 s)
specs/001-pir-ledstrip-fade/quickstart.md           # section 5: kantoorlamp.local
specs/001-pir-ledstrip-fade/contracts/ha-api.md     # host and friendly name after the rename
```

**Structure Decision**: one device YAML, as in 001. The new `number:`, `select:` and
`datetime:` blocks go in section 6 (inputs and Home Assistant entities). The file grows to
roughly 900 lines; splitting into `packages/` is still not needed (YAGNI).

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| V.1: the live on period, fade times, window and reaction mode are no longer read from `substitutions:` but from Home Assistant entities stored on the lamp; the substitutions hold their defaults, ranges and steps | The spec's whole purpose is to change these values without editing the YAML (FR-001, FR-004) | Keeping them as build-time substitutions only is exactly what the owner asked to replace. All numbers in the logic still come from substitutions or from these named entities, so there are no magic numbers |
| Hardware gate: **W16 (b)**, the summer/winter time change, can only be observed on 28 March 2027 | FR-010 asks the window to follow local time across the change | Holding delivery for 4.5 months gains little: ESPHome applies the zone's built-in DST rule (research S6), and W16 (a) proves the local-time computation. W16 (b) is recorded in the quickstart when it happens |
