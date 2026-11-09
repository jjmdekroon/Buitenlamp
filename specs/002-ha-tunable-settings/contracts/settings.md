# Contract: Owner Settings, changes in 1.2.0

**Feature**: [../spec.md](../spec.md) | **Covers**: FR-001, FR-004, FR-005, FR-010 |
**Background**: [../research.md](../research.md) S1–S3, S6–S9

This changes [001 contracts/settings.md](../../001-pir-ledstrip-fade/contracts/settings.md).
Entries not listed here are unchanged.

## What the settings location now means

- The five timing and window values and the reaction mode are now chosen in **Home
  Assistant**. The entries below are only their **defaults**: used until a value is chosen in
  Home Assistant, and ignored after that, also after an update (FR-005). This is the same rule
  as the `on_*` colour defaults of 001.
- Stabilization period, manual-mode fallback time, number of LEDs, channel order, location,
  pins, time servers, log level and names stay build-time settings only.

## `substitutions:`: owner settings, changed and new

| Name | Default | Allowed range | Unit | Replaces | Spec |
|------|---------|---------------|------|----------|------|
| `on_period_s` | `"60"` | 1–86400 | s | `on_period_ms` (`"60000"`) | FR-001, FR-007 |
| `fade_in_s` | `"1.0"` | 0.1–10 | s | `fade_in_ms` (`"1000"`) | FR-001, FR-008 |
| `fade_out_s` | `"5.0"` | 0.1–60 | s | `fade_out_ms` (`"5000"`) | FR-001, FR-008 |
| `window_start_default` | `"00:00:00"` | a valid time of day | `HH:MM:SS` | new | FR-001, FR-010 |
| `window_end_default` | `"00:00:00"` | a valid time of day | `HH:MM:SS` | new | FR-001, FR-010 |
| `reaction_mode_default` | `"only when dark"` | one of the four options in [ha-api.md](ha-api.md) | — | new | FR-001, FR-011 |
| `timezone` | `"Europe/Amsterdam"` | an IANA zone name; **must be the same zone as Home Assistant's**, which the lamp uses while connected (research S6) | — | new (was the build PC's zone) | FR-010 |

**Converting a 1.1.0 configuration**: divide `on_period_ms`, `fade_in_ms` and `fade_out_ms` by
1000 and enter them as `on_period_s`, `fade_in_s` and `fade_out_s`.

**Range checks**: the three numbers keep their `static_assert` in `on_boot`, now comparing
against the range constants below, so a default out of range stops the build. ESPHome itself
checks the time format, that `reaction_mode_default` is one of the options, and the zone name.

## `substitutions:`: fixed constants, new

| Name | Value | Why fixed |
|------|-------|-----------|
| `on_period_min_s`, `on_period_max_s`, `on_period_step_s` | `"1"`, `"86400"`, `"1"` | range and step of On period (FR-001) |
| `fade_in_min_s`, `fade_in_max_s` | `"0.1"`, `"10"` | range of Fade-in time (FR-001) |
| `fade_out_min_s`, `fade_out_max_s` | `"0.1"`, `"60"` | range of Fade-out time (FR-001) |
| `fade_step_s` | `"0.1"` | step of both fade times (FR-001) |
| `rm_dark`, `rm_window`, `rm_window_and_dark`, `rm_always` | `"0"`, `"1"`, `"2"`, `"3"` | indexes of the Reaction mode options, in their fixed order (research S3) |
| `on_period_now_ms` | `"((uint32_t) lroundf(id(on_period).state * 1000.0f))"` | shared expression: the current On period in ms (research S7) |
| `fade_in_now_ms` | `"((uint32_t) lroundf(id(fade_in_time).state * 1000.0f))"` | shared expression: the current Fade-in time in ms |
| `fade_out_now_ms` | `"((uint32_t) lroundf(id(fade_out_time).state * 1000.0f))"` | shared expression: the current Fade-out time in ms |
| `flash_write_interval` | `"500ms"` | how often ESPHome writes changed values to flash (`preferences:`). 500 ms keeps every change within the 2 s of FR-004, also brightness and colour, which are checked once a second (research S8). A longer value breaks FR-004 |

## Secrets

Unchanged.
