# Research: Lamp Settings Adjustable from Home Assistant

**Feature**: [spec.md](spec.md) | **Plan**: [plan.md](plan.md) | **Date**: 2026-11-09

This builds on [001 research](../001-pir-ledstrip-fade/research.md) (R1–R18), which stays valid
except where noted. Items here are numbered S1–S11 to keep them apart from 001's R-numbers.
Statements marked *(verified in source)* were checked against ESPHome 2026.10.1 as installed in
`.venv/`.

**Revised 2026-11-09** after the spec's clarifications on power cuts and option names:
- S8 now stores every change within 2 s (`flash_write_interval` 500 ms). This replaces the
  60 s save of 001 R18 for brightness and colour too.
- S6 sets the time zone on both time sources.
- S1 notes what happens to a value between steps.
- S10: the owner accepts visible steps at the lowest levels at the end of fade-outs longer than
  10 s (FR-009).

## S1. Timing settings as Home Assistant number entities

**Decision**:
- Three `number: platform: template` entities: **On period**, **Fade-in time** and **Fade-out
  time**, with `optimistic: true`, `restore_value: true`, `unit_of_measurement: s`,
  `entity_category: config` and `mode: box`.
- Ranges and steps (FR-001) are fixed constants in `substitutions:`:

  | Entity | min | max | step | default (owner setting) |
  |--------|-----|-----|------|-------------------------|
  | On period | 1 | 86400 | 1 | `on_period_s` = 60 |
  | Fade-in time | 0.1 | 10 | 0.1 | `fade_in_s` = 1.0 |
  | Fade-out time | 0.1 | 60 | 0.1 | `fade_out_s` = 5.0 |

- The owner settings `fade_in_ms`, `on_period_ms` and `fade_out_ms` are **renamed** to
  `fade_in_s`, `on_period_s` and `fade_out_s`, in seconds. They are now only the
  `initial_value` of the entities, so they get the same unit and range the owner sees in Home
  Assistant. The `static_assert` range checks in `on_boot` move to the new names and compare
  against the same min/max constants, so there is one definition of each range.
- Out-of-range values are refused by ESPHome itself: `NumberCall::perform()` logs a warning
  and returns without calling `control()` when the value is below `min_value` or above
  `max_value` *(verified in source: `number/number_call.cpp`)*. Home Assistant's number card
  also limits input to the range. This covers FR-002.
- Neither ESPHome nor Home Assistant enforces the step for a value sent by an automation, so a
  value between steps (for example On period 90.5 s) is accepted and used as it is. FR-002 only
  refuses values out of range, so this is allowed; the card itself offers only whole steps.
- `mode: box` because a slider from 1 to 86400 s cannot be set to an exact value; the fades use
  the same mode for consistency.
- `entity_category: config` puts the entities in the **Configuration** section of the device
  page, apart from the status entities.

**Rationale**: a template number with `optimistic` and `restore_value` is ESPHome's standard
way to expose a stored, owner-editable value; it needs no lambda and no `set_action`.

**Alternatives considered**:
- *Keep the substitutions in ms and convert with a Jinja expression*
  (`${ fade_in_ms | float / 1000 }`): ESPHome 2026.10 supports Jinja in substitutions
  *(verified in source: `substitutions/jinja.py`)*, but the owner would then edit milliseconds
  in the YAML and see seconds in Home Assistant for the same value.
- *Entities in milliseconds*: exact, but "60000" for a minute is unfriendly and the spec states
  the ranges in seconds.
- *Globals plus a Home Assistant `input_number` helper and an automation*: puts the lamp's
  behaviour in Home Assistant, breaks FR-006 when Home Assistant is down at boot, and needs
  setup outside the device.

## S2. Time window as Home Assistant time entities

**Decision**:
- Two `datetime: platform: template` entities with `type: time`: **Time window start** and
  **Time window end**, `optimistic: true`, `restore_value: true`, `entity_category: config`,
  `initial_value` from the owner settings `window_start_default` and `window_end_default`
  (both `"00:00:00"`).
- The lamp uses only hour and minute (`id(window_start).hour`, `.minute`; verified in
  `datetime/time_entity.h`). Seconds that Home Assistant's time picker sends are stored but
  ignored, which gives the minute precision of FR-001.
- **Inside the window** at local time t (minutes since midnight), with start S and end E:
  - S = E: always inside (no restriction, FR-010);
  - S < E: S ≤ t < E;
  - S > E (crosses midnight): t ≥ S or t < E.
  The end minute itself is outside, so 08:00–17:00 covers 08:00 up to and including 16:59, and
  two windows 08:00–17:00 and 17:00–08:00 never overlap.

**Rationale**: the time entity gives Home Assistant a native time picker and stores itself
*(verified in source: `template/datetime/template_time.cpp`, `make_entity_preference`)*.

**Alternatives considered**:
- *Two numbers (hour and minute) per edge*: four entities and an awkward card.
- *A text entity "08:00-17:00"*: needs parsing and string handling in a lambda (constitution
  III.1) and gives no input validation.

## S3. Reaction mode as a Home Assistant select

**Decision**:
- One `select: platform: template` entity, **Reaction mode**, `optimistic: true`,
  `restore_value: true`, `entity_category: config`, with the options, in this order:
  `only when dark`, `only in time window`, `in time window and dark`, `always`.
  `initial_option: ${reaction_mode_default}`, default `only when dark` (behaviour of 001).
- The lamp reads `id(reaction_mode).active_index()` *(verified in source:
  `select/select.h`)*, not the option text, so no string is compared or built in a loop
  (constitution III.1). The four indexes are named fixed constants in `substitutions:`
  (`rm_dark` 0, `rm_window` 1, `rm_window_and_dark` 2, `rm_always` 3), so the logic has no
  magic numbers.
- **The order of the options is a contract**: ESPHome stores the selected *index*, so
  reordering or inserting options would change a stored choice after an update. New options
  may only be appended.

**Alternatives considered**:
- *Two switches ("use darkness", "use time window")*: four combinations, but "neither" would
  have to mean "always", which is not obvious on a card; the owner chose one setting with four
  named choices (spec clarification 2026-11-09).

## S4. One place that decides whether motion counts: `motion_allowed`

**Decision**:
- A new internal template binary sensor `motion_allowed` holds the conditions of FR-011 and
  FR-014 in one short lambda:
  - time unknown (Daylight or In time window has no state) → true (FR-014);
  - otherwise by reaction mode: dark; inside the window; both; or always true.
- `motion_started` checks `motion_allowed` instead of Daylight. `fallback_timer` checks it
  instead of Daylight (FR-015).
- `motion_allowed`'s `on_release` (conditions stop being met) runs the script
  **`conditions_ended`**, which is 001's `dark_period_ended` renamed: if a counted motion is in
  progress it stops counting and the on period starts (FR-012). It therefore fires for every
  way the conditions can end: the window ends, the sun rises in a mode that uses darkness, or
  the owner changes the mode or the window.
- `daylight` loses its `on_press` trigger; it is now only a status entity and an input of
  `motion_allowed`.
- Motion still counts only on its rising edge, so a motion that started while the conditions
  were not met never counts when they start being met (FR-013).

**Behaviour change against 001 (accepted)**: ESPHome fires no `on_press`/`on_release` for a
binary sensor's first state, and `daylight`'s first state was "light" when the time became known
in daytime, so in 001 a counted motion from the unknown-time period simply ran on. Now
`motion_allowed` starts **true** (time unknown) and drops to false when the time becomes known
in daytime, so `conditions_ended` runs and the light fades out one on period later. That is
still "a cycle already running finishes normally" (001/FR-023).

**Rationale**: constitution IV.2 asks for one place per decision. With four modes, repeating the
combination in `motion_started` and `fallback_timer` would duplicate it.

**Alternatives considered**:
- *A lambda condition in each script*: two copies of the mode logic.
- *Exposing `motion_allowed` to Home Assistant*: not required by the spec (YAGNI); In time
  window and Daylight together already explain why the lamp does or does not react.

## S5. The "In time window" status entity

**Decision**:
- A template binary sensor **In time window** (`in_window`), shown in Home Assistant (FR-017).
- Its lambda returns no state while `id(sntp_time).now()` is not valid (time unknown), and
  otherwise the rule of S2.
- A template binary sensor evaluates its lambda on every main-loop pass and publishes only
  changes *(verified in source: `template_binary_sensor.cpp`)*. `now()` converts the system
  time with the pre-parsed POSIX zone rule; no allocation and well under 10 ms (constitution
  II.1, III.1). The status therefore changes within one loop pass after a minute boundary or a
  settings change, far inside the 2 s of FR-017 and the 1 min of SC-006.

**Alternatives considered**:
- *An `interval` that publishes the state*: needs an extra trigger on every window change to
  meet the 2 s limit.
- *ESPHome `on_time` cron triggers*: fixed at build time, so they cannot follow a window chosen
  in Home Assistant.

## S6. Local time and time zone

**Decision**:
- Add the owner setting `timezone: "Europe/Amsterdam"` and set `timezone: ${timezone}` on
  **both** time sources, `sntp_time` and `ha_time`.
  - A time source without `timezone` uses the time zone of the PC that builds the firmware
    *(verified in source: `time/__init__.py`, `detect_tz`)*.
  - Each time source installs its own zone as the one global zone (`setup_time_core_` runs once
    per source and calls `set_global_tz`; *verified in source*), so the last one in the
    generated code wins. Setting the zone on `sntp_time` alone would let `ha_time` replace it
    with the build PC's zone.
- ESPHome turns the zone into a POSIX rule at build time, including the summer/winter time
  switch. 08:00 therefore stays 08:00 local time across the change (spec edge case).
- Home Assistant 2026.3 and newer also send their own parsed time zone over the API, which the
  lamp applies *(verified in source: `api/api_connection.cpp`)*. While Home Assistant is
  connected, its zone wins; without it, `timezone` applies. `timezone` must therefore name the
  same zone as Home Assistant (contracts/settings.md), so the window does not move when Home
  Assistant connects.
- Darkness keeps following the sun's elevation and does not use the zone (001 R7).

## S7. When new values take effect

**Decision**:
- **On period**: `on_period_timer`'s `delay` becomes a lambda that reads the On period entity.
  ESPHome evaluates a templated delay once, when the delay starts, so a running on period keeps
  its length and the next one uses the new value (FR-003).
- **Fades**: `lamp_fade_on` and `lamp_fade_off` compute `fade_length_ms` from the Fade-in or
  Fade-out entity when they start; the light's running transition is not touched (FR-003).
  The partial-fade rule of 001 R6 is unchanged (FR-008).
- **Home Assistant's own colour changes** use the light's `default_transition_length`
  (001 R18). The Fade-in time entity's `on_value` calls
  `id(strip).set_default_transition_length(ms)` *(verified in source: `light/light_state.h`)*,
  so they follow the chosen fade-in time too (FR-008 via 001/FR-027). `on_value` also fires
  for the value published at setup, so the restored value is applied at every boot.
- The three conversions from seconds to milliseconds are shared expressions in
  `substitutions:` (`fade_in_now_ms`, `fade_out_now_ms`, `on_period_now_ms`), next to 001's
  `strip_level`, so each is written once (constitution IV.2).

## S8. Persistence and flash writes

**Decision**:
- `restore_value: true` on the six entities (FR-004).
- A new `preferences:` block with `flash_write_interval: ${flash_write_interval}`, a fixed
  constant of **500 ms**, instead of ESPHome's default of 60 s. This meets FR-004's rule that a
  change is stored within 2 seconds, for the six settings and for 001's brightness and colour
  (clarification 2026-11-09).

**How a change reaches flash** *(verified in source: `esp32/preferences.cpp`,
`preferences/syncer.h`, `globals/globals_component.h`, `globals/__init__.py`,
`template/number/template_number.cpp`, `template/select/template_select.h`,
`template/datetime/template_time.cpp`)*:
1. **The six settings**: the entity's `control()` queues the save at once, under a key derived
   from the entity (`make_entity_preference`).
2. **Brightness and colour** (001's `chosen_*` globals): a restoring global is a polling
   component that compares its value with the last one it checked every `update_interval`
   (default 1 s) and queues a save when it differs.
3. **Every `flash_write_interval`**, ESPHome writes the queued saves. The ESP32 backend writes a
   key only when its content differs from what is in flash (`is_changed_`), and returns at once
   when nothing is queued.

**Worst-case time from change to flash**:

| Value | Queued after | Written after | Total |
|-------|--------------|---------------|-------|
| The six settings | 0 | ≤ 500 ms | ≤ 0.5 s |
| Brightness and colour | ≤ 1 s | ≤ 500 ms | ≤ 1.5 s |

Writing to NVS (the ESP32's key-value store in flash) takes milliseconds, so both stay inside
2 s with margin.

**Flash wear** (constitution III.3):
- Writes still happen only when a value changes, and at most once per value per interval.
- The six settings change only when the owner enters a value: the numbers use `mode: box` (no
  slider), and the time and select entities change in single steps.
- Brightness and colour are checked once a second, so even dragging a colour wheel causes at most
  six writes per second.
- The NVS partition is 448 KB (`IDF_NVS_SIZE = 0x70000`, *verified in source:
  `esp32/__init__.py`*), and NVS spreads writes over all its pages. At 100,000 erase cycles per
  page that allows hundreds of millions of small writes: decades at any realistic use.

**Other effects**:
- With nothing queued, the sync every 500 ms only checks an empty list: no flash access and no
  measurable load.
- At `DEBUG`, every sync that writes something logs `Writing N items: … written` (tag
  `preferences`). The quickstart uses this line to see when a change reached flash, and that
  nothing is written while nothing changes.
- ESPHome's own safe-mode boot counter is also kept in flash, which is the default on ESP32. It
  is written at every boot and cleared about 1 minute later (`boot_is_good_after`, default
  1 min), which logs one `Writing 1 items` line at that moment *(verified in source:
  `safe_mode/safe_mode.cpp`, `preferences.py`)*. This was already so in 001 and is not a
  setting change; the quickstart's idle check starts after it.
- Before a restart for an update, ESPHome runs its shutdown hooks, which store the globals and
  flush pending saves *(verified in source: `core/application.cpp`)*.
- A power cut during a write: NVS writes the new entry before it erases the old one, so after
  the restart the lamp has either the old or the new value, never a damaged one (ESP-IDF NVS
  documentation, power-loss resilience).
- Renaming one of the six entities changes its storage key, so the stored value would be lost
  and the default used again. The names are part of the contract
  ([contracts/ha-api.md](contracts/ha-api.md)).

**Alternatives considered**:
- *Keep 60 s* (the first version of this plan): a power cut within a minute of a change loses
  it. The spec no longer allows this.
- *Save only the six settings immediately, with a lambda that calls
  `global_preferences->sync()` after each change*: the entity queues its save after `on_value`
  runs (`publish_state` comes before `pref_.save`), so the sync would have to wait one loop
  pass. It would also leave brightness and colour at 60 s, which the clarification ruled out.
- *`flash_write_interval: 1s`*: brightness and colour could take up to 2 s (1 s poll + 1 s
  sync), with no margin.

## S9. Defaults and their precedence (FR-005)

- An entity uses its `initial_value` only when nothing is stored for it (S1, S2, S3). After the
  owner has chosen a value in Home Assistant, a new default in the YAML has no effect, also
  after an update, as with 001's `on_*` defaults.
- A default out of range stops the build: `static_assert` for the three numbers;
  ESPHome's own validation for the time format and for `initial_option` (it must be one of the
  options; *verified in source: `template/select/__init__.py`*).

## S10. Smoothness at the extremes of the fade range (FR-009)

- The SK6812 has 8 bits per channel and ESPHome does no temporal dithering. With
  `gamma_correct: 2.8`, the lowest output steps are coarse, so on a 60 s fade-out the last
  seconds pass through a few discrete levels. 001 already allowed a 60 s fade-out through the
  settings, so this is not new, but FR-009 now makes it a requirement.
- At 0.1 s the fade has about 10 frames at the 100 Hz refresh limit; it looks like a quick but
  not instant change.
- **Decision** (owner, 2026-11-09): no design change. Visible steps between the lowest levels at
  the end of fade-outs longer than 10 s are accepted and written into FR-009; flicker is still
  not allowed. Quickstart scenario W4 checks both extremes and records what is seen at 60 s.
- **Alternative rejected**: lowering the maximum fade-out. It would make values that were valid
  in 1.1.0 invalid (spec assumption on ranges), and the steps come from the strip's 8 bits per
  channel, which no setting can remove.

## S11. Memory and compatibility

- **Memory**: three more component types (`number`, `select`, `datetime`) and seven entities.
  Estimate +15–30 KB flash and well under 1 KB static RAM, against 47.0 % flash and 34.2 % RAM
  measured for 1.1.0. Measured at the resource gate.
- **Home Assistant compatibility**: only new entities, nothing renamed, so `project: version`
  becomes **1.2.0** (minor). Existing dashboards and automations keep working.
- **Owner settings compatibility**: `fade_in_ms`, `on_period_ms` and `fade_out_ms` are renamed
  to seconds (S1). An owner who changed them in 1.1.0 carries the values over by hand (divide
  by 1000); the README and [contracts/settings.md](contracts/settings.md) describe this.
  Because the values become Home Assistant defaults, this only matters until a value is chosen
  in Home Assistant.
