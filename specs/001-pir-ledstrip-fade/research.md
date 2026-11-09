# Research: PIR-Controlled LED Strip with Fade (ESPHome)

**Feature**: [spec.md](spec.md) | **Plan**: [plan.md](plan.md) | **Date**: 2026-11-01

This replaces the earlier research for the hand-written Arduino design. The project now builds
ESPHome firmware (constitution v2.0.0). Each section records one decision, why it was made, and
what else was considered.

Facts marked *(verified in source)* were read from the ESPHome 2026.10.1 package downloaded from
PyPI. Facts marked *(verified locally)* were read on the development PC.

## R1. Build tool, version and framework

**Decision**:
- Use **ESPHome 2026.10.1**, the newest release on PyPI on 2026-11-01 *(verified locally)*.
- Pin it in `requirements.txt` (`esphome==2026.10.1`) and install it into a project virtual
  environment `.venv/` with the PC's Python 3.13.15 *(verified locally)*.
- The YAML also sets `esphome: min_version: 2026.10.1`.
- Framework: **esp-idf**. It is ESPHome's default for every ESP32 variant *(verified in source:
  `_set_default_framework`)*, so nothing has to be overridden.
- ESPHome builds with its own esp-idf toolchain. The older PlatformIO toolchain is deprecated
  and will be removed in 2027.2.0 *(verified in source)*. The first compile downloads ESP-IDF
  once, which takes several minutes.

**Rationale**:
- A project virtual environment keeps the exact version with the repository and away from any
  other Python tools on the PC.
- esp-idf is what ESPHome tests and optimises first. The Arduino framework would add nothing
  this lamp needs.

**Alternatives considered**:
- *ESPHome Device Builder add-on on the Pi*: rejected by the owner, because the YAML would live
  outside this repository.
- *Arduino framework*: works on the ESP32-C3, but brings no benefit here.
- *An older ESPHome*: the LED strip keys changed in 2026.10. Starting on the current release
  avoids a migration straight away.

## R2. Board and pin assignment

**Decision**:
- Use `esp32: variant: esp32c3`, without a `board` key. The `board` key is no longer
  recommended, and `variant` alone assumes a standard Espressif devkit with 4 MB flash.
- **PIR signal: GPIO4** (XIAO pin D2).
- **LED data: GPIO5** (XIAO pin D3).
- Both pins are substitutions, so another board only needs those two values changed.

**Rationale**:
- GPIO4 and GPIO5 exist on every common ESP32-C3 board: DevKitM-1, XIAO ESP32C3 and the C3
  SuperMini.
- They avoid the pins that cause trouble: the strapping pins (GPIO2, 8 and 9), the USB pins
  (18 and 19) and the flash pins (11–17).
- The wiring is unchanged from the earlier design.

**Alternatives considered**:
- *GPIO10 for data*: available, but on the XIAO it is shared with SPI MOSI.
- *GPIO2, 8 or 9*: rejected. A strapping pin pulled by an external circuit can stop the board
  from booting.

## R3. LED strip driver

**Decision**: use `light: platform: esp32_rmt_led_strip` with:

| Key | Value | Why |
|-----|-------|-----|
| `chipset` | `SK6812` | supported value *(verified in source)* |
| `channel_colors` | `${led_channel_order}`, default `GRBW` | replaces the deprecated `rgb_order` / `is_rgbw` keys in 2026.10 *(verified in source)* |
| `use_psram` | `false` | the ESP32-C3 has no PSRAM, and the default is `true` |
| `restore_mode` | `ALWAYS_OFF` | off at every boot and never saved to flash *(verified in source)*. The default changed in 2026.10.0, so it must be set explicitly |
| `internal` | `true` | Home Assistant must not get brightness or colour controls (FR-027) |
| `max_refresh_rate` | `${led_max_refresh}`, default `10ms` | at most 100 frames per second |

`num_leds` is capped at **240**. One SK6812 RGBW pixel takes 32 bits × 1.25 µs = 40 µs, so
240 pixels take 9.6 ms, which fits inside the 10 ms refresh slot. The RMT peripheral sends the
data in the background.

**Alternatives considered**:
- *`neopixelbus` platform*: offers more options than this lamp needs.
- *FastLED*: no RGBW support.
- *A refresh every 5 s against noise on the data line* (part of the earlier design): left out
  under YAGNI. The level shifter and a short data wire are the real protection (R4).

## R4. Level shifting and strip power

The hardware is unchanged from the earlier design.

**Decision**:
- Drive the strip's data input through a **74AHCT125** (or 74AHCT1G125) buffer powered from
  5 V, with a **330 Ω** series resistor at the strip's DIN.
- Fit a **10 kΩ pull-down** on GPIO5.
- Fit a **1000 µF / ≥10 V** capacitor across the strip's 5 V and GND at the feed point.
- The strip, the ESP32-C3 board (through its 5 V pin) and the PIR share one GND.
- Size the 5 V supply at `NUM_LEDS × 20 mA × 1.2 + 0.3 A`, with a fuse matched to the supply.

**Rationale**:
- SK6812 needs a logic-high input of at least 0.7 × VDD = 3.5 V. The ESP32-C3 outputs 3.3 V,
  which is marginal and becomes unreliable in the cold or on a long strip.
- The pull-down keeps the data line low while the board boots, so the strip does not latch
  random colours (FR-009).
- The capacitor absorbs the inrush current when the supply is switched on.

**Alternatives considered**:
- *No level shifter*: works on the bench, then fails intermittently outdoors.
- *A "sacrificial" first pixel*: hard to explain and maintain.

## R5. Motion sensor: Seeed Studio Grove mini PIR (S16-L221D)

**Findings** (unchanged): the sensor runs on 3.3–5 V, reaches up to 2 m, and its SIG output
goes HIGH on motion. Its hold time, retrigger behaviour and warm-up time are **not published**.

**Decision**:
- Power the PIR from **3.3 V**.
- Read SIG with `binary_sensor: platform: gpio` on GPIO4, pin mode input with `pulldown: true`,
  so a disconnected sensor reads "no motion" (FR-012). This sensor is internal (`pir_raw`).
- Filter both edges with `delayed_on` and `delayed_off` of `${pir_min_pulse_ms}` (50 ms), so
  shorter signals never count (FR-011).
- **Stabilization (FR-010)**: a global `pir_ready` starts false and is set by an `on_boot`
  automation after `delay: ${pir_stabilization_ms}ms`. This uses ESPHome's scheduler, so there
  is no `millis()` arithmetic to get wrong.
- The **Motion** entity Home Assistant sees is a template binary sensor equal to
  `pir_ready AND pir_raw`. Motion during stabilization is therefore neither reported nor acted on.
- The quickstart measures the real hold time and warm-up time.

**Risk to flag**: 2 m is short for a driveway. The standard Grove PIR (101020020, up to 6 m)
has the same 3-wire interface and can be swapped in without configuration changes.

## R6. Fades and the brightness curve

**How ESPHome fades**:
- A transition's progress is linear in time. ESPHome eases it with the smootherstep curve
  `6x⁵ − 15x⁴ + 10x³` *(verified in source: `light_transformer.h`, `smoothed_progress`)*.
- Gamma correction (default exponent 2.8) is applied per channel to
  state × brightness × channel *(verified in source)*.
- A new transition always starts from the light's current, partly faded values *(verified in
  source: `start_transition_` uses `current_values`)*. Reversing a fade therefore never jumps
  (FR-006).

**Decision**:
- Use the built-in transitions with `gamma_correct: ${led_gamma}` (2.8).
- Gamma 2.8 is close to how the eye sees brightness (the CIE lightness curve), so equal steps
  in ESPHome's brightness look like equal steps of light.
- Smootherstep makes each fade start and end gently, with most of the change in the middle.
  This meets FR-007: no visible steps, no jump at the start, and the change is not concentrated
  at the start or end. V3 and V7 confirm it by eye.
- **Normal rate from a partial level** (FR-006, FR-027): `transition_length` is a duration, not
  a rate, so the fade scripts work out the length with a lambda:
  - level = `${strip_level}`, a substitution holding
    `current_values.get_state() × current_values.get_brightness()`, from 0 to 1;
  - fade-in length = `fade_in_ms × (1 − level)`;
  - fade-out length = `fade_out_ms × level`.

  A full fade therefore takes exactly 1 s or 5 s, and a partial one proportionally less.

  *Updated 2026-11-02 (R18):* "full" is now the chosen brightness B (`chosen_brightness`), so
  the lengths are measured against B instead of 1:
  - fade-in length = `fade_in_ms × min(1, |B − level| / B)`;
  - fade-out length = `fade_out_ms × min(1, level / B)`.

  A fade from off to B therefore still takes 1 s and a fade from B to off 5 s, whatever B is.
  The `min(1, …)` caps the case where the level is above a newly chosen lower B.
- **On-colour (FR-017)**: `light.turn_on` sets `red`, `green`, `blue` and `white` from the
  `on_*` substitutions (fractions 0–1), with `brightness: 100%` and `color_brightness: 100%`.
  - Gamma raises every channel to the same power, so the ratio between channels stays the same
    throughout a fade.
  - The on-colour values themselves are on ESPHome's 0–1 brightness scale.
- **Fully off (FR-008)**: at the end of a turn-off transition the state is 0, so every channel
  is 0.

**Alternatives considered**:
- *Our own fade loop* that calls the light every 10 ms: floods the log and the scheduler.
- *An `addressable_lambda` effect with an exact CIE curve*: more code, and no difference anyone
  could see at these fade lengths.
- *A linear fade without gamma*: looks as if it jumps on and then crawls, so FR-007 fails.

## R7. Darkness

**Decision**:
- Use `sun:` with `latitude: ${latitude}`, `longitude: ${longitude}` and `time_id: sntp_time`.
- Read the sun's elevation with `sensor: platform: sun, type: elevation`, internal, with
  `update_interval: ${daylight_check_interval}` (10 s; the default is 60 s).
- The **Daylight** entity is a template binary sensor (`device_class: light`):
  - **no state** while the elevation is not a number (NaN);
  - **ON** when elevation ≥ `${sun_horizon_deg}` (−0.83333°);
  - **OFF** below that.
- Before the time is known, ESPHome's sun returns NaN *(verified in source: `sun.cpp` returns
  NaN for an invalid date)*. So the entity has no state, Home Assistant shows "unknown", and the
  logic treats it as unknown (react day and night, FR-023).
- Once the time is known it stays known until the next power-up.
- −0.83333° is ESPHome's own `DEFAULT_ELEVATION` *(verified in source)*. It is the standard
  sunrise/sunset definition (the sun's upper edge at the horizon, including refraction), so the
  dark period matches published times (FR-019).
- It is a fixed constant, not an owner setting. The spec rules out an offset.

**Rationale**:
- Comparing elevation directly needs no time zone, no daylight-saving logic and no midnight
  handling, so summer and winter time cannot shift it (FR-021).
- A single Daylight entity is both the status Home Assistant shows and the input the logic
  reads, so the dark rule is written once.

**Alternatives considered**:
- *`on_sunset` / `on_sunrise` triggers*: they only fire on the crossing, so a lamp that boots in
  the middle of the night would not know it is dark.
- *`sun.is_below_horizon` conditions*: correct, but they would repeat the threshold in several
  places.

## R8. Time

**Decision** (clarified: both sources):
- Configure two time sources:
  - `time: platform: sntp` with `id: sntp_time` and servers `${ntp_server_1}` and
    `${ntp_server_2}`;
  - `time: platform: homeassistant` with `id: ha_time`.
- Both set the one shared system clock with `settimeofday` *(verified in source:
  `real_time_clock.cpp`)*. Whichever answers first makes the time known, and each re-syncs every
  15 min while it can be reached *(verified in source: default `update_interval` 15min)*. This
  meets FR-020's "at least every 24 hours".
- **Retry while unknown (FR-020)**: an `interval: 60s` checks the condition `time.has_time`. If
  the time is not known yet, it runs `component.update` on both time sources.
- **Accuracy**: between syncs the ESP32-C3 crystal (±10 ppm) drifts under 1 s per day.
- **SC-011**: with both sources and the 60 s retry, the lamp learns the time well within 2 min
  of WiFi returning.

**Alternatives considered**:
- *SNTP only*: the time stays unknown after a power cut while the internet is down, even with
  Home Assistant running.
- *Home Assistant only*: darkness would depend on Home Assistant, against FR-028.

## R9. Home Assistant integration

> **Superseded in part on 2026-11-02 (R18)**: the owner now wants to choose brightness and
> colour in Home Assistant. The Buitenlamp entity is therefore the strip light itself, not a
> template switch. The rest of this section (API, discovery, reporting, offline detection)
> still applies; "Why a switch, not a light" is kept as the record of the earlier decision.

**Decision**:
- Use the native API with `encryption: key: !secret api_encryption_key` and
  `reboot_timeout: 0s`.
- Home Assistant finds the lamp over mDNS and offers it as a discovered device. The owner clicks
  **Add** and pastes the key once (clarified).

The device exposes these entities ([contracts/ha-api.md](contracts/ha-api.md)):

| Entity | Component | Notes |
|--------|-----------|-------|
| Buitenlamp | `switch: platform: template` | `name: None`, so it takes the device name; `icon: mdi:lightbulb` |
| Motion | template binary sensor | `device_class: motion` |
| Daylight | template binary sensor | `device_class: light` |
| Brightness | template sensor | % |
| Mode | template text sensor | `automatic` / `manual` |

**Why a switch, not a light** (clarified with the owner):
- A template switch runs `turn_on_action` / `turn_off_action` on **every** command, even when
  the state does not change *(verified in source: `Switch::turn_on` always calls
  `write_state`, and `TemplateSwitch::write_state` always fires the trigger)*. An "on" while the
  light is already on from motion therefore still switches to manual mode, as the spec requires.
- The switch's state comes from a lambda: the strip's target state, `remote_values.is_on()`.
  It is ON while fading in or on, and OFF while fading out or off.
- `restore_mode: DISABLED` is required. Any other mode makes the switch act on its restored
  state at boot and fire `turn_off_action` *(verified in source: `TemplateSwitch::setup`)*.

**Reporting (FR-025)**:
- ESPHome pushes entity changes to Home Assistant as soon as they happen.
- Brightness is a template sensor with `update_interval: 1s` and a `delta` filter, so it reports
  at most once a second and only when the value changes.
- Mode is a template text sensor with no lambda. Scripts publish it only when the mode changes,
  so no text is built over and over (Principle III).

**Commands (FR-027)**: they run as soon as the API receives them, well within 1 s.

**Offline detection (FR-026)**:
- Home Assistant's client library sends a ping every 20 s and closes the connection after 90 s
  without an answer (`KEEP_ALIVE_FREQUENCY = 20.0`, `KEEP_ALIVE_TIMEOUT_RATIO = 4.5` in
  aioesphomeapi).
- After a silent power loss, Home Assistant therefore marks the device unavailable after
  90–110 s. The spec was changed to "within 2 minutes" (clarified).

**Alternatives considered**:
- *MQTT discovery*: the owner chose the native API.
- *Exposing the strip light itself*: Home Assistant would get brightness and colour controls,
  and its "on" could not be told apart from a motion "on".

## R10. Home Assistant reachability and the manual-mode fallback

**Decision**:
- Home Assistant counts as **reachable** while at least one API client that subscribes to
  entity states is connected: the condition `api.connected` with
  `state_subscription_only: true` *(verified in source)*.
- `api: on_client_disconnected`: if no such client is left, run the script `fallback_timer`.
  It uses `mode: restart` and `delay: ${manual_fallback_ms}ms`. This trigger fires after the
  client has been removed, so the condition sees only the clients that are left *(verified in
  source, 2026.10.1: `APIServer::remove_client_`)*.
- `api: on_client_connected` only logs. It fires while the client's Hello is handled, before
  the client subscribes to states, so the condition is still false there *(verified in source,
  2026.10.1: `APIConnection::complete_authentication_` fires the trigger, and
  `on_subscribe_states_request` sets the flag later)*. Stopping the timer from this trigger
  would therefore never work.
- Instead, `fallback_timer` checks the condition when it expires. If Home Assistant is back,
  nothing happens. If it is not and the lamp is in manual mode, the FR-029 rule applies
  (data-model T10).
- A short interruption therefore changes nothing (SC-015). Every new disconnect restarts the
  timer, so the fallback needs `${manual_fallback_ms}` without interruption.

**Caveat**: since 2026.4.0, `esphome logs` also subscribes to states, so it counts as a client.
While testing the fallback, run it with `--no-states` *(verified in source: the flag exists)*.

## R11. Lamp logic as ESPHome scripts (Principles II and IV)

**Decision**:
- The mode (automatic or manual) is held in the global `manual_mode`.
- The light's target (on or off) and its running fade belong to the strip's light component.
- Together they give three named states, AutoOff, AutoOn and ManualOn
  ([data-model.md](data-model.md)).
- Every transition is a named script, and each script's comment lists the requirements it
  implements:

| Script | What it does |
|--------|--------------|
| `lamp_fade_on`, `lamp_fade_off` | the only places that call the light; they work out the fade length (R6) |
| `motion_started` | the single place where "motion counts" is decided |
| `motion_ended` | clears `need_fresh_motion`; restarts the on-period timer after counted motion |
| `on_period_timer` | `mode: restart`, `delay: ${on_period_ms}ms`, then fades out in automatic mode |
| `manual_on`, `manual_off` | handle Home Assistant commands |
| `fallback_timer` | see R10 |
| `dark_period_ended` | at sunrise, ends a counted motion that is still going and starts the on period (FR-018) |

**How motion counts**:
- Motion counts on its **rising edge**, when it is not daylight (dark or unknown) and
  `need_fresh_motion` is false.
- A motion that starts in daylight never counts, even if it lasts into the dark. This matches
  the spec's edge case: the lamp turns on at the next detection after dark.
- A counted motion that is still going at sunrise stops counting there: `dark_period_ended`
  starts the on period, so the light fades out one on period after sunrise even if someone
  keeps moving (FR-018).
- The on period counts from the **end** of the counted motion, as in the earlier design. The
  sensor's own hold time simply adds to it.

**Timing with no `millis()` arithmetic**:
- All waiting uses script `delay`s on ESPHome's scheduler, which keeps time as a 64-bit
  millisecond count *(verified in source: `scheduler.cpp`, `millis_64`)*. The 49.7-day rollover
  of the 32-bit counter therefore cannot shorten or stretch any timer (FR-015).
- The only `millis()` calls are in debug log lines.

**Alternatives considered**:
- *Comparing `millis()` timestamps held in globals*: works, but needs careful unsigned
  arithmetic in several lambdas, which is exactly what the scripts avoid.
- *A `select` holding the state name*: adds an entity and string comparisons without making the
  logic clearer.

## R12. Updates over WiFi and safe mode

**Decision**:
- Use `ota: - platform: esphome` with `encryption:` and no key of its own. It takes the API's
  encryption key *(verified in source: `_resolve_encryption_key`)*.
  - ESPHome warns that an OTA `password` next to an API key "wastes significant flash and RAM",
    and recommends encryption instead.
  - The spec's "update password" (FR-030) is therefore the device's encryption key: one secret,
    kept in `secrets.yaml`.
- A wrong key fails the encrypted handshake, so the upload is refused and the lamp carries on.
- During an accepted upload, the main loop is busy receiving the image, so the strip holds its
  current output. The device then restarts and comes up off (FR-030, FR-009).
- **Safe mode** is added automatically with OTA. After 10 boots in a row that fail before
  running for 1 minute, only WiFi and OTA start, so a broken update can be replaced over WiFi.
  Only the first load needs USB.

**Risk recorded in Complexity Tracking**: in safe mode the light component is not started. If
the strip was lit before a crash, it keeps its last frame until a power cycle.

## R13. Watchdog, reboots and network loss

**Decision**:
- Keep the ESP32 task watchdog at ESPHome's default of 5 s (`esp32: watchdog_timeout`, allowed
  range 5–60 s, *verified in source*). It is never disabled.
- Set `wifi: reboot_timeout: 0s` and `api: reboot_timeout: 0s`. Both default to 15 min, and a
  reboot turns the light off, against FR-014 and FR-028 (constitution: Outdoor operation).
- Configure no WiFi fallback access point and no captive portal (YAGNI). If the WiFi password
  changes, the owner reloads over USB, or over WiFi before the old network disappears.
- Leave the brownout detector at the framework default (on).

**Risk**: the ESPHome WiFi page notes that the IP stack sometimes needs a full reboot to
recover. If that shows up in the field, the remedy is a restart that only happens while the
lamp is off and in automatic mode. It is not built now (YAGNI).

## R14. Validation without host-side tests

**Decision** (constitution v2.0.0):
- `esphome config` checks the schema.
- `esphome compile` checks the lambdas. An `on_boot` lambda also holds `static_assert`s on
  every numeric substitution, so an out-of-range setting fails the build with a clear message
  ([contracts/settings.md](contracts/settings.md)).
- Behaviour is proven by the on-device scenarios in [quickstart.md](quickstart.md). Its
  traceability table maps every FR to a YAML element and to at least one scenario.
- Debug log lines (R17) print `millis()` timestamps, so the timing scenarios can be read from
  the log.

## R15. Memory and flash budget

**Decision**:
- Use ESPHome's default partition layout for 4 MB, with two app slots of 1.75 MB each for OTA.
- `esphome compile` prints `RAM: … %` and `Flash: … %` *(verified in source:
  `espidf/size_summary.py`)*. `esphome analyze-memory` gives a breakdown.
- Expected: flash about 50–60 % of the app slot; static RAM about 15–25 %. Both are measured at
  the resource gate.
- Measured (2026-11-01, `num_leds` 60, INFO): flash 47.0 %, static RAM 34.0 %. RAM is above
  the estimate but well within the 75 % limit; 31.4 % of it is already used by the platform
  alone (WiFi, API, OTA, before any lamp logic).

## R16. Security

**Decision**:
- `secrets.yaml` is git-ignored and holds `wifi_ssid`, `wifi_password` and
  `api_encryption_key`. `secrets.example.yaml` is committed.
- The key is 32 random bytes in base64, generated on the PC.
- One key protects both the API and OTA (R12).
- No web server, no access point and no plain-text password are configured.
- All traffic stays in the home network.

## R17. Logging

**Decision**:
- `logger: level: ${log_level}`, with `INFO` in delivered firmware and `DEBUG` during
  validation (constitution: Logging).
- The scripts log each event at DEBUG with `millis()`: motion on/off, counted or ignored, fade
  start with its length, fade complete, on-period end, mode changes and fallback.
- At INFO these lines are compiled out.
- Watch the log with `esphome logs esphome/buitenlamp.yaml --no-states` (R10).

## R18. Brightness and colour from Home Assistant (2026-11-02)

**Decision**:
- Expose the strip light `strip` to Home Assistant (`name: None`, so it takes the device name
  Buitenlamp and the object id `buitenlamp`; no longer `internal`). Home Assistant then shows
  one light card with on/off, brightness and an RGBW colour control. The template switch
  `lamp_switch` is removed.
- `default_transition_length: ${fade_in_ms}ms` on the strip, so a brightness or colour change
  from Home Assistant fades instead of jumping.
- **Telling Home Assistant commands apart from the lamp's own light calls**:
  - Every light call, from the API or from an action, runs `LightCall::perform()`, which
    calls `publish_state()` and so fires the light's `on_state` trigger synchronously, also
    when nothing changes *(verified in source, 2026.10.1: `light_call.cpp`, `light_state.cpp`)*.
  - The light publishes nothing else: publishing during a transition is opt-in
    (`transition_state_publish_interval`) and not supported on addressable lights *(verified
    in source: `light/__init__.py`)*.
  - `lamp_fade_on` and `lamp_fade_off` set the global `lamp_call_active` just before their
    light call and clear it just after. `on_state` runs `ha_command` only when the flag is
    clear, so anything it sees is a command from Home Assistant.
  - `LightState::setup()` performs one call at boot *(verified in source)*. The flag therefore
    starts **true**, and `on_boot` clears it by running `lamp_fade_off` once.
- **What a command means** (`ha_command`):
  - target on (a plain "on", or a new brightness or colour, which Home Assistant always sends
    as "on"): copy the strip's target values into the `chosen_*` globals, then `manual_on`
    (manual mode, and `lamp_fade_on` re-issues the fade at the normal rate);
  - target off: `manual_off`.
  - Home Assistant's call has already started its own fade; the re-issued call starts from the
    current values, so nothing jumps. When the level is already at B, `lamp_fade_on` skips
    its call and Home Assistant's colour fade (the default transition) finishes on its own.
- **The on-setting** is six float globals: `chosen_brightness`, `chosen_color_brightness`,
  `chosen_red`, `chosen_green`, `chosen_blue`, `chosen_white`. Their initial values come from
  the substitutions `on_brightness`, `on_red`, `on_green`, `on_blue` and `on_white`
  (colour brightness starts at 1.0). `lamp_fade_on` turns the strip on with them and
  `lamp_fade_off` turns it off with them, so the strip's target values always hold the
  on-setting and a plain "on" from Home Assistant uses it.
- **Remembering it (FR-031)**: the `chosen_*` globals use `restore_value: yes`. ESPHome saves a
  restoring global only when its value has changed (`memcmp` against the last saved value),
  and writes flash at most once per `flash_write_interval` (default 60 s) *(verified in
  source: `globals_component.h`, `preferences/__init__.py`)*. Motion never changes them, so
  only choices in Home Assistant cause writes.
  *Superseded in part on 2026-11-09 (002 research S8)*: from 1.2.0, `flash_write_interval` is
  500 ms, so a chosen brightness and colour is in flash within 1.5 s instead of within 60 s.
  See [002 research.md](../002-ha-tunable-settings/research.md).

**Alternatives considered**:
- *Keep the switch and add a brightness `number` and a colour `select` with presets*: keeps
  the 1.0.0 entities, but gives no free colour choice; rejected by the owner.
- *A virtual light (template outputs) next to the switch*: its own on/off would mean nothing
  and confuse the card.
- *`restore_mode: RESTORE_AND_OFF` on the strip*: restores the colour, but the light then saves
  its whole state, including on/off, at every motion cycle, against constitution III.3.
- *Only "on" means manual, a colour change keeps the mode*: needs the previous target state,
  which is already overwritten when `on_state` runs; more state for little gain. Every "on"
  from Home Assistant therefore means manual mode, as before.

**Power**: with a free colour choice, all four channels can be at full at once, about four
times the white-only current. The supply sizing in
[contracts/hardware.md](contracts/hardware.md) changes to `num_leds × 0.080 A × 1.2 + 0.3 A`.
