# Data Model: PIR-Controlled LED Strip with Fade (ESPHome)

**Feature**: [spec.md](spec.md) | **Plan**: [plan.md](plan.md) | **Date**: 2026-11-01

Everything lives in one ESPHome configuration, `esphome/buitenlamp.yaml`. The lamp's state is
never saved to flash:
- the strip uses `restore_mode: ALWAYS_OFF`;
- every state global uses `restore_value: no`.

After power-up or any restart every state item starts from its initial value (FR-009,
FR-030). The only saved values are the on-setting, the `chosen_*` globals (FR-031, updated
2026-11-02).

All waiting is done with script `delay`s on ESPHome's 64-bit scheduler (research R11). No
lambda compares `millis()` values (FR-015).

## Settings (`substitutions:`)

The full list, with defaults, ranges and units, is in
[contracts/settings.md](contracts/settings.md). Every number used by an automation, script or
lambda comes from a substitution. Range checks are compile-time `static_assert`s in one
`on_boot` lambda.

## Globals

| id | Type | Initial | Meaning | Spec |
|----|------|---------|---------|------|
| `manual_mode` | `bool` | `false` | true after an "on" from Home Assistant, until "off" or the fallback | FR-027, FR-029 |
| `need_fresh_motion` | `bool` | `false` | set by "off" while motion is active; cleared when motion ends | FR-027 |
| `pir_ready` | `bool` | `false` | false during the stabilization period after boot | FR-010 |
| `motion_counted` | `bool` | `false` | the current motion pulse counted for automatic switching | FR-018, FR-029 |
| `fade_length_ms` | `uint32_t` | `0` | length of the fade being started; set and read only by `lamp_fade_on` and `lamp_fade_off`, so one calculation serves the log line and the light call | FR-006, FR-027 |
| `lamp_call_active` | `bool` | **`true`** | true while `lamp_fade_on` or `lamp_fade_off` calls the light, so the strip's `on_state` can tell those calls from Home Assistant commands. Starts true so the light's own call at boot is ignored; `on_boot` clears it through `lamp_fade_off` (research R18) | FR-027 |

**On-setting** (`restore_value: yes`, saved only when changed; FR-017, FR-031):

| id | Type | Initial | Meaning |
|----|------|---------|---------|
| `chosen_brightness` | `float` | `${on_brightness}` | brightness B of the on-setting, 0.01–1 |
| `chosen_color_brightness` | `float` | `1.0` | brightness of the RGB part relative to white, as Home Assistant sends it |
| `chosen_red`, `chosen_green`, `chosen_blue`, `chosen_white` | `float` | `${on_red}`, `${on_green}`, `${on_blue}`, `${on_white}` | the colour, 0–1 per channel |

## Entities

| id | Component / platform | Home Assistant name | Internal | Content |
|----|----------------------|---------------------|----------|---------|
| `strip` | `light` / `esp32_rmt_led_strip` | Buitenlamp (device name) | no | the SK6812 RGBW strip; owns the fades; on/off, brightness and colour in Home Assistant; `on_state` → `ha_command` for Home Assistant commands |
| `pir_raw` | `binary_sensor` / `gpio` | — | yes | PIR SIG, pull-down, 50 ms `delayed_on` and `delayed_off` |
| `motion` | `binary_sensor` / `template` | Motion | no | `pir_ready && pir_raw.state` |
| `sun_elevation` | `sensor` / `sun` (elevation) | — | yes | degrees, every `${daylight_check_interval}`; NaN while the time is unknown |
| `daylight` | `binary_sensor` / `template` | Daylight | no | no state = unknown; ON = light; OFF = dark |
| `brightness` | `sensor` / `template` | Brightness | no | `round(${strip_level} × 100)` %, every 1 s, only when it changes |
| `mode_text` | `text_sensor` / `template` | Mode | no | `automatic` / `manual`, published by scripts |
| `sntp_time` | `time` / `sntp` | — | — | internet time |
| `ha_time` | `time` / `homeassistant` | — | — | Home Assistant time; shares the system clock |

### Daylight: one source for the dark rule

| Daylight state | Condition | Logic reads it as |
|----------------|-----------|-------------------|
| no state | `sun_elevation` is NaN (time not yet known since power-up) | **unknown**: motion counts day and night (FR-023) |
| OFF | elevation < `${sun_horizon_deg}` (−0.83333°) | **dark**: motion counts (FR-018) |
| ON | elevation ≥ `${sun_horizon_deg}` | **light**: motion does not count |

## Lamp state machine

### States

The lamp's state is the pair (`manual_mode`, the strip's target on/off). Whether a fade is
running belongs to the light component.

| State | `manual_mode` | Strip target | Covers |
|-------|---------------|--------------|--------|
| **AutoOff** | false | off | off, or fading out |
| **AutoOn** | false | on | fading in, or holding full brightness |
| **ManualOn** | true | on | switched on from Home Assistant, fading in or holding |

After power-up the state is AutoOff, the level is 0 and `pir_ready` is false (T12).

### When motion counts

This is decided only in the script `motion_started`, on the rising edge of `motion`. Motion
counts when **both** of these hold:
- Daylight is not ON (so it is dark, or the time is unknown); FR-018, FR-023;
- `need_fresh_motion` is false; FR-027.

`motion` itself is already false during stabilization (FR-010), and a disconnected sensor reads
LOW (FR-012).

### Fade lengths

Only the scripts `lamp_fade_on` and `lamp_fade_off` call the light:
- level = `${strip_level}`, from 0 to 1. This substitution holds the one definition,
  `strip.current_values.get_state() × get_brightness()`, which the Brightness sensor also uses;
- B = `chosen_brightness`, the full level of the on-setting (updated 2026-11-02, research R18).
- `lamp_fade_on`: `light.turn_on` with the on-setting (`chosen_brightness`,
  `chosen_color_brightness`, `chosen_red`, `chosen_green`, `chosen_blue`, `chosen_white`) and
  `transition_length = ${fade_in_ms} × min(1, |B − level| / B)` ms. It does nothing if the
  target is already on and that length is 0 (FR-002, FR-006, FR-017).
- `lamp_fade_off`: `light.control` with `state: false`, the on-setting and
  `transition_length = ${fade_out_ms} × min(1, level / B)` ms. Passing the on-setting keeps it
  in the strip's target values, so a plain "on" from Home Assistant uses it (FR-005, FR-008,
  FR-027, FR-031).
- Both set `lamp_call_active` around their light call (research R18).
- Each script (`mode: restart`) logs the start, waits for the transition length, then logs
  "fade complete". A newer fade cancels the pending log line.

### Transitions

| # | From | Trigger | To | Done by | Spec |
|---|------|---------|----|---------|------|
| T1 | AutoOff, level 0 | motion counts | AutoOn | `motion_started` → `lamp_fade_on` | FR-002 |
| T2 | AutoOn, fading in | transition ends | AutoOn, holding | light component | FR-002 |
| T3 | AutoOn | motion counts | AutoOn; `on_period_timer` stopped while the motion lasts; a running fade-in is not restarted | `motion_started` | FR-004 |
| T3b | AutoOn or ManualOn | counted motion ends | same; `on_period_timer` (re)started | `motion_ended` | FR-003, FR-004 |
| T4 | AutoOn | `on_period_timer` expires | AutoOff (fade-out) | `on_period_timer` → `lamp_fade_off` | FR-003, FR-005, FR-018 |
| T5 | AutoOff, fading out | motion counts | AutoOn, fade up from the current level | `motion_started` → `lamp_fade_on` | FR-006 |
| T6 | AutoOff, fading out | transition ends | AutoOff, level 0 | light component | FR-008 |
| T7 | any | "on" from Home Assistant, including a new brightness or colour (strip `on_state`, target on) | ManualOn; the strip's target values become the on-setting | `ha_command` (copies them into `chosen_*`) → `manual_on` → `lamp_fade_on` | FR-027, FR-031 |
| T8 | ManualOn, fading in | transition ends | ManualOn, holding | light component | FR-027 |
| T9 | any | "off" from Home Assistant (strip `on_state`, target off) | AutoOff | `ha_command` → `manual_off` → `lamp_fade_off`; `need_fresh_motion = motion.state`; `motion_counted = false`; `on_period_timer` stopped | FR-027 |
| T10 | ManualOn | `fallback_timer` expires and no Home Assistant client is connected (unreachable for `${manual_fallback_ms}` without interruption) | AutoOn if Daylight is not ON **and** (`motion_counted` or `on_period_timer` is running); otherwise AutoOff | `fallback_timer` → `lamp_fade_off` when needed | FR-029 |
| T11 | any | last state-subscribed client gone | same; `fallback_timer` (re)started | `api: on_client_disconnected` | FR-029 |
| T12 | power-up, reset or update | — | AutoOff, level 0, `pir_ready` false; `pir_ready` set after `${pir_stabilization_ms}`; the on-setting restored | `restore_mode` and global initial values; `on_boot` runs `lamp_fade_off` (loads the on-setting into the strip, clears `lamp_call_active`), then the delay | FR-009, FR-010, FR-030, FR-031 |
| T13 | AutoOn or ManualOn, counted motion in progress | Daylight turns ON (sunrise) | same; motion no longer counted; `on_period_timer` started | `daylight` `on_press` → `dark_period_ended` | FR-018 |

**Details per script**:
- `motion_started`, in ManualOn: it still records the motion (`motion_counted`, stops
  `on_period_timer`) but does not touch the light. The motion is still reported (Story 4,
  scenario 6) and the FR-029 fallback can use it.
- `motion_ended`: always sets `need_fresh_motion = false`. If `motion_counted` is set, it clears
  it and runs `on_period_timer`.
- `on_period_timer` expiring in ManualOn changes nothing. It only means "no recent motion" for
  T10.
- `dark_period_ended`: if `motion_counted` is set, it clears it and runs `on_period_timer`. A
  motion still going at sunrise therefore stops counting, and in AutoOn the light fades out one
  on period after sunrise (FR-018). In ManualOn the timer's expiry changes nothing.
- `manual_on` and `manual_off` publish the new Mode, and `fallback_timer` does too when it
  returns to automatic.

**Why `need_fresh_motion` exists (T9)**: after an "off", a PIR that is still HIGH because
someone is standing there must not switch the light straight back on. The flag is set only
while motion is active, and is cleared when that motion ends, so the next *new* detection works
as usual.

## Status report (FR-025)

| What the spec asks for | Entity | Value |
|------------------------|--------|-------|
| online / offline | (API connection) | Home Assistant marks the device unavailable when the connection is lost |
| light on/off | Buitenlamp light | ON while the strip's target is on (fading in or on) |
| chosen brightness and colour | Buitenlamp light (brightness, RGBW colour) | the on-setting |
| current brightness | Brightness | 0–100 % of full scale, follows the fades |
| motion detected | Motion | ON / OFF (also in daylight and in manual mode) |
| dark / daylight | Daylight | OFF (dark) / ON (light) / unknown |
| automatic / manual | Mode | `automatic` / `manual` |

## Timers

| Timer | Mechanism | Length | Restart behaviour |
|-------|-----------|--------|-------------------|
| Stabilization | `esphome: on_boot` → `delay` → `pir_ready = true` | `${pir_stabilization_ms}` | once per boot |
| On period | script `on_period_timer`, `mode: restart` | `${on_period_ms}` | restarted at the end of each counted motion |
| Manual fallback | script `fallback_timer`, `mode: restart` | `${manual_fallback_ms}` | restarted when the last Home Assistant client disconnects; at expiry it acts only if none has reconnected |
| Time retry | `interval: 60s` + condition `time.has_time` | 60 s | runs until the time is known |
| Fade complete log | inside `lamp_fade_on` / `lamp_fade_off` | the computed fade length | cancelled by the next fade |
