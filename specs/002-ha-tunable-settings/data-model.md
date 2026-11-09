# Data Model: Lamp Settings Adjustable from Home Assistant

**Feature**: [spec.md](spec.md) | **Plan**: [plan.md](plan.md) | **Date**: 2026-11-09

This is a delta on [001 data-model](../001-pir-ledstrip-fade/data-model.md). Everything there
stays valid unless this file changes it. Requirement ids without a prefix are from this feature's
spec; "001/FR-0xx" refers to feature 001. (In the YAML's comments it is the other way round:
001's ids have no prefix and this feature's ids are written `002/FR-0xx`; plan.md, IV.3.)

## Settings (`substitutions:`)

Changed and new entries are listed in [contracts/settings.md](contracts/settings.md). In short:
- `fade_in_ms`, `on_period_ms` and `fade_out_ms` become `fade_in_s`, `on_period_s` and
  `fade_out_s` (seconds), and are now the **defaults** of the Home Assistant entities below.
- New owner settings: `window_start_default`, `window_end_default`, `reaction_mode_default`,
  `timezone`.
- New fixed constants: the ranges and steps of the three numbers, the four reaction-mode
  indexes, three shared expressions that convert the current settings to milliseconds, and
  `flash_write_interval` (500 ms).

## New stored entities (owner settings in Home Assistant)

All six use `optimistic: true`, `restore_value: true` and `entity_category: config`. They are
the only new values written to flash (constitution III.3, FR-004).

**When a value is stored** (FR-004, research S8):

| Stored value | Written to flash | Within |
|--------------|------------------|--------|
| The six entities below | when the value changes; queued at once, written at the next `flash_write_interval` | 0.5 s |
| 001's `chosen_*` globals (brightness and colour, 001/FR-031) | when the value changes; checked every 1 s, written at the next `flash_write_interval` | 1.5 s |

A value that did not change is never written. A power cut during a write leaves the old or the
new value, never a damaged one.

| id | Component / platform | Home Assistant name | Values | Default | Read by | Spec |
|----|----------------------|---------------------|--------|---------|---------|------|
| `on_period` | `number` / `template` | On period | 1–86400 s, step 1 | `${on_period_s}` (60) | `on_period_timer` (via `${on_period_now_ms}`) | FR-001, FR-007 |
| `fade_in_time` | `number` / `template` | Fade-in time | 0.1–10 s, step 0.1 | `${fade_in_s}` (1.0) | `lamp_fade_on` (via `${fade_in_now_ms}`); its `on_value` sets the strip's default transition | FR-001, FR-008 |
| `fade_out_time` | `number` / `template` | Fade-out time | 0.1–60 s, step 0.1 | `${fade_out_s}` (5.0) | `lamp_fade_off` (via `${fade_out_now_ms}`) | FR-001, FR-008 |
| `window_start` | `datetime` / `template`, `type: time` | Time window start | time of day (hour and minute used) | `${window_start_default}` (00:00) | `in_window` | FR-001, FR-010 |
| `window_end` | `datetime` / `template`, `type: time` | Time window end | time of day (hour and minute used) | `${window_end_default}` (00:00) | `in_window` | FR-001, FR-010 |
| `reaction_mode` | `select` / `template` | Reaction mode | `only when dark` (index 0), `only in time window` (1), `in time window and dark` (2), `always` (3) | `${reaction_mode_default}` (`only when dark`) | `motion_allowed` | FR-001, FR-011 |

**Validation** (FR-002): ESPHome refuses a number outside min/max and keeps the old value; a
time entity only accepts valid times; a select only accepts its own options (research S1–S3).
A number between steps, which only an automation can send, is accepted as it is (research S1).

## New derived entities

| id | Component / platform | Home Assistant name | Internal | Content | Spec |
|----|----------------------|---------------------|----------|---------|------|
| `in_window` | `binary_sensor` / `template` | In time window | no | no state while the time is unknown; otherwise the window rule below | FR-010, FR-017 |
| `motion_allowed` | `binary_sensor` / `template` | — | yes | whether motion may switch the lamp on now; table below. `on_release` → `conditions_ended` | FR-011, FR-012, FR-014 |

### Window rule (`in_window`)

With t, S and E the local time, start and end in minutes since midnight (hour × 60 + minute):

| Case | Inside when |
|------|-------------|
| S = E | always (no restriction) |
| S < E | S ≤ t < E |
| S > E (crosses midnight) | t ≥ S or t < E |

### When motion is allowed (`motion_allowed`)

| Time known? | Reaction mode | `motion_allowed` |
|-------------|---------------|------------------|
| no (Daylight or In time window has no state) | any | **ON** (FR-014) |
| yes | only when dark | Daylight is OFF |
| yes | only in time window | In time window is ON |
| yes | in time window and dark | In time window is ON and Daylight is OFF |
| yes | always | **ON** |

`motion_allowed` starts ON at boot (time unknown). Its first state fires no trigger.

## Changed scripts

| Script | Change | Spec |
|--------|--------|------|
| `motion_started` | "Daylight not ON" is replaced by "`motion_allowed` is ON"; the ignore log names the reason (`not allowed` or `fresh motion needed`) | FR-011, FR-013 |
| `conditions_ended` | **renamed** from `dark_period_ended`; triggered by `motion_allowed` `on_release` instead of `daylight` `on_press`; same body | FR-012 |
| `on_period_timer` | `delay` reads `${on_period_now_ms}` when it starts | FR-003, FR-007 |
| `lamp_fade_on` | full fade length is `${fade_in_now_ms}` instead of `${fade_in_ms}` | FR-003, FR-008 |
| `lamp_fade_off` | full fade length is `${fade_out_now_ms}` instead of `${fade_out_ms}` | FR-003, FR-008 |
| `fallback_timer` | "Daylight not ON" is replaced by "`motion_allowed` is ON" | FR-015 |

`daylight` keeps its lambda and stays in Home Assistant, but loses its `on_press` trigger.

`strip` keeps `default_transition_length: ${fade_in_s}s` as its build-time value; the Fade-in
time entity's `on_value` replaces it at runtime with the chosen value, also at boot (research
S7).

## Lamp state machine: changed transitions

States (AutoOff, AutoOn, ManualOn) are unchanged. "Motion counts" now means: rising edge of
`motion`, `motion_allowed` ON and `need_fresh_motion` false.

| # | Change against 001 | Spec |
|---|--------------------|------|
| T1, T3, T5 | "motion counts" uses `motion_allowed` instead of Daylight | FR-011 |
| T4 | the on period length is the On period entity's value when the timer started | FR-007, FR-003 |
| T10 | AutoOn if `motion_allowed` is ON **and** (`motion_counted` or `on_period_timer` running); otherwise AutoOff | FR-015 |
| T13 | trigger is `motion_allowed` turning OFF (window ends, sunrise in a mode that uses darkness, mode or window changed, or the time becomes known while not allowed); script `conditions_ended` | FR-012 |
| T14 (new) | any state; a setting is changed in Home Assistant → the state stays the same; the value is stored within 0.5 s; a running fade or on period keeps its length; for the Fade-in time the strip's default transition is updated | FR-003, FR-004 |

Changing the reaction mode or window can switch `motion_allowed` off, which is T13; switching it
on does nothing until the next motion (FR-013).

## Status report (additions to 001's FR-025 table)

| What | Entity | Value |
|------|--------|-------|
| current settings | On period, Fade-in time, Fade-out time, Time window start, Time window end, Reaction mode | the stored values; confirmed within 2 s of a change (FR-003) |
| inside the time window | In time window | ON / OFF / unknown (FR-017) |

## Log lines (additions and changes, DEBUG, tag `lamp`)

As built in 1.2.0. Every line starts with `t=<millis>`.

| Line | When | Change against 1.1.0 |
|------|------|----------------------|
| `on period <s> s` | On period gets a value, also the restored one at boot | new |
| `fade-in time <s> s` | Fade-in time gets a value, also at boot | new |
| `fade-out time <s> s` | Fade-out time gets a value, also at boot | new |
| `window start <HH:MM>`, `window end <HH:MM>` | a window time gets a value, also at boot | new |
| `reaction mode <option>` | Reaction mode gets a value, also at boot | new |
| `motion ON, ignored (not allowed)` | motion while `motion_allowed` is OFF | was `(daylight)` |
| `conditions ended, motion no longer counted` | `conditions_ended` ends a counted motion | was `dark period ended, …` |
| `stays on (recent motion, motion allowed)` | the manual-mode fallback keeps the light on | was `(recent motion in the dark)` |

ESPHome itself logs `[preferences] Writing N items: … written` whenever it writes to flash
(research S8); the quickstart's power-cut trials use it.

## Timers (changes)

| Timer | Change |
|-------|--------|
| On period | length read from the On period entity when the timer starts (was `${on_period_ms}`) |
| Fade complete log | length from the Fade-in or Fade-out entity |
