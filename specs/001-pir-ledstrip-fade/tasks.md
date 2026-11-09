---

description: "Task list for the PIR-controlled LED strip with fade (ESPHome)"
---

# Tasks: PIR-Controlled LED Strip with Fade

**Input**: Design documents from `/specs/001-pir-ledstrip-fade/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/ (ha-api.md,
settings.md, hardware.md), quickstart.md

**Tests**: there are no automated tests, because constitution v2.0.0 drops host-side tests.
Each story ends with two checks instead:
- the compile gate (`esphome config` and `esphome compile`);
- the on-device scenarios from quickstart.md (V1–V28) that prove the story.

**Organization**: tasks are grouped by user story. Almost every task edits the same file,
`esphome/buitenlamp.yaml`, so those tasks run in order. Most of the parallel work is hardware and
documentation.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: can run in parallel (different files, no dependencies)
- **[Story]**: the user story from spec.md (US1–US5)
- **Hands-on**: needs a person with the hardware, Home Assistant or private values. An agent can
  prepare these tasks and explain the steps, but cannot do them.

## Conventions (apply to every task)

**Paths** are from the repository root. Commands run in PowerShell from the repository root and
use `.venv\Scripts\esphome`.

**Layout of `esphome/buitenlamp.yaml`**. This follows plan.md, with two changes that ESPHome
requires:

1. `substitutions:`: owner settings, then fixed constants
2. `esphome:` (including `on_boot:`), `esp32:`, `logger:`, `wifi:`, `api:`, `ota:`
3. `time:`, `sun:`
4. `light:` (the strip)
5. `globals:`, `script:`, `interval:`
6. `binary_sensor:`, `sensor:`, `switch:`, `text_sensor:`

- ESPHome only accepts `on_boot:` inside `esphome:`. The boot logic therefore sits in section 2,
  with a comment pointing to section 5.
- ESPHome rejects a top-level key that appears twice. `binary_sensor:` and `sensor:` therefore
  each appear once, in section 6.
  - Internal inputs (`pir_raw`, `sun_elevation`) come first, under a `# Internal inputs` comment.
  - The Home Assistant entities follow, under `# Home Assistant entities`.

**Requirement labels**: every script and automation gets a comment naming the requirements it
implements, for example `# FR-003, FR-004` (constitution IV.3). Each task gives the ids.

**Numbers**: every tunable comes from a substitution (constitution V.1). The only literals
allowed are unit conversions (`100.0f` for percent), the full level `1.0f` and `0` for "no time
left".

**Log lines** (research R17):
- Use `logger.log` with `level: DEBUG` and `tag: lamp`.
- Start every message with `t=%lu`, and pass `(unsigned long) millis()` as the first argument.
- On ESP-IDF 5, `uint32_t` is `unsigned long`, so `%u` gives a format warning and fails the
  compile gate. Print `uint32_t` globals the same way, with `%lu` and a cast.

**Compile gate and upload**: the last implementation task of every story runs these commands:

```powershell
.venv\Scripts\esphome config esphome/buitenlamp.yaml
.venv\Scripts\esphome compile esphome/buitenlamp.yaml
.venv\Scripts\esphome upload esphome/buitenlamp.yaml --device buitenlamp.local
.venv\Scripts\esphome logs esphome/buitenlamp.yaml --device buitenlamp.local --no-states
```

- `esphome config` must end with "Configuration is valid!".
- `esphome compile` must show no warnings that point to `esphome/buitenlamp.yaml`.
- Record each scenario's result in the results table in section 9 of
  `specs/001-pir-ledstrip-fade/quickstart.md` (the "Scenarios V1–V28 passed" row).
- Run each scenario as many times as the Trials table in section 8 of quickstart.md says.

**If `esphome config` rejects a key named in a task**, look up the current name in the ESPHome
2026.10.1 documentation, use it, and note the change in `specs/001-pir-ledstrip-fade/research.md`.

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: the build tool, the secrets and the bench hardware.

- [X] T001 [P] Create `.gitignore` at the repository root with three entries, each with a short
  comment: `esphome/secrets.yaml` (FR-022), `esphome/.esphome/` (ESPHome build output) and
  `.venv/`.
- [X] T002 [P] Create `requirements.txt` at the repository root with one line:
  `esphome==2026.10.1` (research R1, constitution V.4).
- [X] T003 [P] Create `esphome/secrets.example.yaml` with placeholders for `wifi_ssid`,
  `wifi_password` and `api_encryption_key` (contracts/settings.md).
  - Header comment: copy this file to `esphome/secrets.yaml` and fill it in. Never commit
    `secrets.yaml`.
  - Comment on `api_encryption_key`:
    - it is 32 random bytes in base64 (44 characters);
    - it protects both the Home Assistant connection and updates over WiFi, so it is the spec's
      "update password" (FR-030, research R12);
    - the generation command from contracts/settings.md.
- [X] T004 Create the virtual environment and install ESPHome, as in section 2 of
  `specs/001-pir-ledstrip-fade/quickstart.md`.
  - Run `python -m venv .venv`, then `.venv\Scripts\python -m pip install -r requirements.txt`.
  - Check that `.venv\Scripts\esphome version` prints `Version: 2026.10.1`.
  - Depends on T002.
- [ ] T005 Hands-on: create `esphome/secrets.yaml` from `esphome/secrets.example.yaml`.
  - Generate `api_encryption_key` with the command in contracts/settings.md. An agent may do
    this step.
  - The owner fills in `wifi_ssid` and `wifi_password` for a 2.4 GHz network.
  - Depends on T001 and T003.
  - Status 2026-11-01: `secrets.yaml` created with a generated `api_encryption_key`;
    `wifi_ssid` and `wifi_password` are still placeholders.
- [ ] T006 [P] Hands-on: build the bench hardware as in
  `specs/001-pir-ledstrip-fade/contracts/hardware.md`.
  - PIR VCC goes to 3V3 (never 5 V), and PIR SIG to GPIO4.
  - GPIO5 goes to 74AHCT125 pin 1A, with a 10 kΩ pull-down to GND. Pin 1Y goes through 330 Ω to
    the strip's DIN.
  - The 74AHCT125 runs on 5 V. Tie its unused A inputs to GND and its unused OE pins to 5 V.
  - Fit 1000 µF across the strip's 5V and GND at the feed point.
  - Use a fused 5 V supply rated at least `num_leds × 0.020 A × 1.2 + 0.3 A`, with one common GND.
  - Before powering up, do the two checks in section 1 of
    `specs/001-pir-ledstrip-fade/quickstart.md`.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: a device that builds, boots, joins WiFi, takes updates over WiFi and is known to
Home Assistant. It has no lamp logic yet.

**⚠️ CRITICAL**: no user story work can begin until this phase is complete.

- [X] T007 Create `esphome/buitenlamp.yaml` with a header comment and section 1, the
  `substitutions:` block (contracts/settings.md, FR-013).
  - The header comment says what the device is, gives the section order from Conventions, and
    points to `specs/001-pir-ledstrip-fade/`.
  - The first part is headed "Owner settings". Every value is a quoted string. Each has a comment
    saying what it controls, its unit and its allowed range:
    - `fade_in_ms: "1000"`: fade-in time, ms, 100–10000 (FR-002)
    - `on_period_ms: "60000"`: on period after the last motion, ms, 1000–86400000 (FR-003)
    - `fade_out_ms: "5000"`: fade-out time, ms, 100–60000 (FR-005)
    - `pir_stabilization_ms: "30000"`: motion ignored after power-up, ms, 0–120000 (FR-010)
    - `pir_min_pulse_ms: "50"`: shortest signal that counts as motion, ms, 1–1000 (FR-011)
    - `manual_fallback_ms: "600000"`: time without Home Assistant before manual mode ends, ms,
      60000–86400000 (FR-029)
    - `num_leds: "60"`: pixels on the strip, 1–240 (research R3)
    - `led_channel_order: "GRBW"`: the strip's channel order, any order of R, G, B and W
    - `on_red: "0.0"`, `on_green: "0.0"`, `on_blue: "0.0"`, `on_white: "1.0"`: the on-colour,
      fraction of full, 0.0–1.0 each and not all 0 (FR-017)
    - `latitude`, `longitude`: an example value, with the comment
      `# MUST be set to the installation location`; degrees, north and east positive (FR-019)
    - `pin_pir: "GPIO4"`, `pin_led_data: "GPIO5"`: not GPIO2, 8, 9 or 11–19, and not the same pin
      (contracts/hardware.md)
    - `ntp_server_1: "pool.ntp.org"`, `ntp_server_2: "time.cloudflare.com"`: internet time
      servers (FR-020)
    - `log_level: "INFO"`: `INFO` when delivered, `DEBUG` for validation only
    - `device_name: "buitenlamp"`, `friendly_name: "Buitenlamp"`: host name and Home Assistant
      name (FR-024)
  - The second part is headed "Fixed constants (not owner settings)". Its comment says that
    changing these changes behaviour the spec fixes. Each value carries the reason it is fixed,
    from contracts/settings.md:
    - `sun_horizon_deg: "-0.83333"`
    - `led_gamma: "2.8"`
    - `led_max_refresh: "10ms"`
    - `daylight_check_interval: "10s"`
    - `brightness_report_interval: "1s"`
    - `time_retry_interval: "60s"`
    - `strip_level: "(id(strip).current_values.get_state() * id(strip).current_values.get_brightness())"`.
      Its comment says it is a shared expression, not a setting: the one definition of the
      strip's current level, 0–1, used by `lamp_fade_on`, `lamp_fade_off` and `brightness`
      (constitution IV.2).
- [X] T008 Add the platform part of section 2 to `esphome/buitenlamp.yaml`.
  - `esphome:` with `name: ${device_name}`, `friendly_name: ${friendly_name}` and
    `min_version: 2026.10.1`.
  - Under `esphome:`, a `project:` with `name: diy.buitenlamp` and `version: "1.0.0"`
    (contracts/ha-api.md).
  - `esp32:` with `variant: esp32c3` and no `board:` key (research R2), plus
    `framework: type: esp-idf` (research R1).
  - A comment in `esp32:` saying that `watchdog_timeout` stays at its 5 s default and is never
    disabled (constitution I.3, research R13).
  - `logger:` with `level: ${log_level}`.
- [X] T009 Add the network part of section 2 to `esphome/buitenlamp.yaml`.
  - `wifi:` with `ssid: !secret wifi_ssid`, `password: !secret wifi_password` and
    `reboot_timeout: 0s`. No `ap:`.
  - `api:` with `encryption: key: !secret api_encryption_key` and `reboot_timeout: 0s`.
  - `ota:` with `- platform: esphome` and an `encryption:` entry that has no key of its own, so it
    reuses the API key (research R12). No `password:` and no `safe_mode:` block.
  - Comment: the reboot timeouts are 0 s because a reboot turns the light off (FR-014, FR-028).
  - Comment: there is no `web_server`, `captive_portal` or access point (constitution V.5).
- [ ] T010 Hands-on: set the real `latitude`, `longitude` and `num_leds` in the `substitutions:`
  block of `esphome/buitenlamp.yaml` (section 3 of `specs/001-pir-ledstrip-fade/quickstart.md`).
  Then set `log_level` to `"DEBUG"` for validation. T059 sets it back.
- [X] T011 Run `esphome config` and `esphome compile` on `esphome/buitenlamp.yaml` and fix any
  errors (section 4 of `specs/001-pir-ledstrip-fade/quickstart.md`). The first compile downloads
  ESP-IDF, which takes several minutes.
- [ ] T012 Hands-on: do the first upload over USB, then show that WiFi updates work (section 5
  of `specs/001-pir-ledstrip-fade/quickstart.md`).
  - Run `.venv\Scripts\esphome upload esphome/buitenlamp.yaml --device COM5`, with the port from
    Device Manager.
  - Check in `esphome logs esphome/buitenlamp.yaml --device COM5` that the board boots and joins
    WiFi.
  - Upload once more with `--device buitenlamp.local`, and open the WiFi log with `--no-states`.
- [ ] T013 Hands-on: add the lamp to Home Assistant (section 6 of
  `specs/001-pir-ledstrip-fade/quickstart.md`).
  - In the "New device discovered: Buitenlamp" notification, click Add and paste
    `api_encryption_key`.
  - Note how long discovery took, and that no address was typed (needed for V13).
  - The device has no entities yet; that is expected. US3 needs this connection for its Home
    Assistant time source.

**Checkpoint**: the board accepts updates over WiFi, logs over WiFi, and is listed in Home
Assistant. Lamp logic can now be added.

---

## Phase 3: User Story 1 - Light fades on when someone approaches and fades off afterwards (Priority: P1) 🎯 MVP

**Goal**: motion fades the strip on in 1 s. It stays on until 60 s after the motion, then fades
off in 5 s. The darkness rule comes in US3, so for now the lamp reacts day and night.

**Independent Test**: this test works at any time of day. After the 30 s stabilization, wave
once and step out of view. The log should show:
- "fade complete" 0.9–1.1 s after the counted `motion ON`;
- the on period ending 60–61 s after `motion OFF`;
- the fade-out complete 4.75–5.25 s after that.

### Implementation for User Story 1

- [X] T014 [US1] Add section 4 to `esphome/buitenlamp.yaml`: `light:` with
  `- platform: esp32_rmt_led_strip` (research R3, R6).
  - `id: strip`, `name: "Strip"` and `internal: true`, so Home Assistant gets no brightness or
    colour controls (FR-027).
  - `pin: ${pin_led_data}`, `num_leds: ${num_leds}`, `chipset: SK6812`,
    `channel_colors: ${led_channel_order}`.
  - `use_psram: false`, because the ESP32-C3 has no PSRAM.
  - `restore_mode: ALWAYS_OFF` (FR-009, constitution I.1). Comment: set explicitly, because the
    default changed in 2026.10.0.
  - `gamma_correct: ${led_gamma}` (FR-007) and `max_refresh_rate: ${led_max_refresh}`.
  - No `effects:`, so the whole strip shows one colour (FR-016).
- [X] T015 [US1] Start the `binary_sensor:` list in section 6 of `esphome/buitenlamp.yaml` with
  `pir_raw`, under `# Internal inputs` (research R5).
  - `platform: gpio`, `id: pir_raw`, `name: "PIR raw"`, `internal: true`.
  - `pin:` with `number: ${pin_pir}` and `mode:` `input: true`, `pulldown: true`. Comment: with
    the pull-down, a disconnected sensor reads "no motion" (FR-012).
  - `filters:` with `- delayed_on: ${pir_min_pulse_ms}ms` and
    `- delayed_off: ${pir_min_pulse_ms}ms` (FR-011, constitution I.5).
  - `on_state:` logs `t=%lu PIR raw %s` with `x ? "ON" : "OFF"`. The sensor check in section 7
    of quickstart.md reads these lines.
  - Comment `# FR-001, FR-011, FR-012`.
- [X] T016 [US1] Add section 5 to `esphome/buitenlamp.yaml`, starting with `globals:` that holds
  `pir_ready` and `motion_counted` (data-model.md, Globals).
  - Both are `type: bool`, `restore_value: no`, `initial_value: 'false'`.
  - Section comment: nothing about the lamp is saved, so every boot starts from these values
    (FR-009, FR-030).
  - `pir_ready` is false during the stabilization period (FR-010).
  - `motion_counted` means the current motion counted for automatic switching (FR-018, FR-029).
- [X] T017 [US1] Add the stabilization period to `esphome: on_boot:` in
  `esphome/buitenlamp.yaml` (data-model T12).
  - Actions, in order: log `t=%lu boot, PIR stabilizing`; `delay: ${pir_stabilization_ms}ms`;
    `globals.set` `pir_ready` to `'true'`; log `t=%lu PIR ready`.
  - Comment `# FR-010`. A second comment says this is lamp logic (section 5) that sits here
    because ESPHome only allows `on_boot` inside `esphome:`.
- [X] T018 [US1] Add `motion` to the section-6 `binary_sensor:` list in
  `esphome/buitenlamp.yaml`, under `# Home Assistant entities` (data-model.md, Entities).
  - `platform: template`, `id: motion`, `name: "Motion"`, `device_class: motion`.
  - `lambda: return id(pir_ready) && id(pir_raw).state;`. Motion during stabilization is
    therefore neither reported nor acted on.
  - `on_press:` runs `- script.execute: motion_started`, and `on_release:` runs
    `- script.execute: motion_ended`.
  - Comment `# FR-001, FR-010, FR-025`.
- [X] T019 [US1] Add `script:` to section 5 of `esphome/buitenlamp.yaml`, with the scripts
  `lamp_fade_on` and `lamp_fade_off`.
  - These are the only two places that call the light (constitution IV.2).
  - This first version always uses the full fade length. T027 and T028 add fades from a partial
    level.
  - `lamp_fade_on` has `mode: restart` and comment `# FR-002, FR-007, FR-017`. Its actions, in
    order:
    - `script.stop: lamp_fade_off`, so only the newest fade logs "fade complete";
    - log `t=%lu fade on, %d ms` with `${fade_in_ms}`;
    - `light.turn_on:` with `id: strip`, `brightness: 100%`, `color_brightness: 100%`,
      `red: ${on_red}`, `green: ${on_green}`, `blue: ${on_blue}`, `white: ${on_white}` and
      `transition_length: ${fade_in_ms}ms`;
    - `wait_until:` with `condition:` `lambda: return !id(strip).is_transformer_active();`;
    - log `t=%lu fade complete`.
  - If `is_transformer_active()` is not accessible, use a `delay:` of the same fade length
    instead of `wait_until`.
  - `lamp_fade_off` has `mode: restart` and comment `# FR-005, FR-008`. It has the same shape:
    - `script.stop: lamp_fade_on`;
    - log `t=%lu fade off, %d ms` with `${fade_out_ms}`;
    - `light.turn_off:` with `id: strip` and `transition_length: ${fade_out_ms}ms`;
    - wait and log as in `lamp_fade_on`.
- [X] T020 [US1] Add the script `on_period_timer` to `esphome/buitenlamp.yaml` (data-model T4).
  - `mode: restart`.
  - Actions: `delay: ${on_period_ms}ms`; log `t=%lu on period ended`;
    `script.execute: lamp_fade_off`.
  - Comment `# FR-003, FR-005, FR-018`.
  - Comment: a script delay runs on ESPHome's 64-bit scheduler, so the `millis()` rollover
    cannot affect it (FR-015, research R11).
- [X] T021 [US1] Add the script `motion_started` to `esphome/buitenlamp.yaml` (data-model T1,
  T5).
  - Comment: this is the only place where "motion counts" is decided. US3 and US4 add their
    conditions here.
  - Set `motion_counted` to `'true'` with `globals.set`, then log `t=%lu motion ON, counted`.
  - Then an `if:` with `condition:` `lambda: return !id(strip).remote_values.is_on();` and
    `then:` `script.execute: lamp_fade_on`.
    - A fade starts only when the target is off, meaning off or fading out.
    - A running fade-in is therefore never restarted (T3).
  - Comment `# FR-002, FR-006`.
- [X] T022 [US1] Add the script `motion_ended` to `esphome/buitenlamp.yaml` (data-model T3b).
  - Log `t=%lu motion OFF`.
  - Then, `if:` `motion_counted` is true:
    - set `motion_counted` to `'false'`;
    - `script.execute: on_period_timer`;
    - log `t=%lu on period started`.
  - The on period counts from the end of the counted motion (research R11).
  - Comment `# FR-003, FR-004`.
- [ ] T023 [US1] Run the compile gate on `esphome/buitenlamp.yaml` and upload over WiFi (see
  Conventions).
  - Status 2026-11-01: compile gate passed (DEBUG, no warnings); upload still to do.

### Validation for User Story 1

- [ ] T024 [US1] Hands-on: check the sensor as described in section 7 of
  `specs/001-pir-ledstrip-fade/quickstart.md`.
  - Write the measured warm-up time and hold time into the section 9 results table.
  - If the warm-up time is well under 30 s, the owner may lower `pir_stabilization_ms`.
- [ ] T025 [US1] Hands-on: run scenarios V1, V2, V3, V4, V7, V12 and V25 from section 8 of
  `specs/001-pir-ledstrip-fade/quickstart.md`. The log-timed scenarios can run in daylight,
  because the darkness rule is not built yet. Run V7, and the smoothness check of V3, after
  dark: flicker and steps at low brightness only show in the dark (SC-003).

**Checkpoint**: MVP. A motion lamp with smooth fades that works without a network. It still
reacts in daylight.

---

## Phase 4: User Story 2 - Light stays on while someone is still there (Priority: P2)

**Goal**: every detection restarts the on period. Motion during a fade-out brings the light
smoothly back up from its current level, at the fade-in rate.

**Independent Test**:
- Trigger the lamp, then move in front of the sensor every 30 s for 3 min. The light never dims.
- Let the fade-out start and wave halfway through. The light rises smoothly to full within 1 s,
  without going dark first.

### Implementation for User Story 2

- [X] T026 [US2] In the script `motion_started` in `esphome/buitenlamp.yaml`, add
  `script.stop: on_period_timer` straight after `motion_counted` is set (data-model T3).
  - Without it, an on period left over from an earlier motion could run out while this motion
    lasts.
  - Add FR-004 to the script's comment.
- [X] T027 [US2] Make `lamp_fade_on` in `esphome/buitenlamp.yaml` start from the current level at
  the normal rate (research R6; data-model.md, Fade lengths).
  - Add the global `fade_length_ms` to `globals:`: `type: uint32_t`, `restore_value: no`,
    `initial_value: '0'`.
    - Comment: the length of the fade being started. Only `lamp_fade_on` and `lamp_fade_off` set
      and read it.
    - It lets one calculation serve both the log line and the light call.
  - Keep `script.stop: lamp_fade_off` as the first action.
  - Next, `globals.set` `fade_length_ms` to `${fade_in_ms} × (1 − ${strip_level})`, cast to
    `uint32_t`. `${strip_level}` is the strip's current level, from 0 to 1.
  - Wrap the remaining actions in an `if:` that skips them when the target is already on and the
    level is 1 (data-model). The condition is
    `!(id(strip).remote_values.is_on() && id(fade_length_ms) == 0)`.
  - The log line prints `fade_length_ms`, and the light call uses
    `transition_length: !lambda 'return id(fade_length_ms);'`.
  - Add FR-006 to the comment.
- [X] T028 [US2] Give `lamp_fade_off` in `esphome/buitenlamp.yaml` the same pattern, with
  `fade_length_ms` = `${fade_out_ms} × ${strip_level}`. It needs no skip condition.
  - A fade-out that starts from a partial level then runs at the normal rate, for example an
    "off" from Home Assistant during a fade-in (FR-027).
  - Add FR-027 to the comment.
- [ ] T029 [US2] Run the compile gate on `esphome/buitenlamp.yaml` and upload over WiFi.
  - Status 2026-11-01: compile gate passed (DEBUG, no warnings); upload still to do.

### Validation for User Story 2

- [ ] T030 [US2] Hands-on: run V5 (30 min) and V6 from
  `specs/001-pir-ledstrip-fade/quickstart.md`. Then repeat V3 and V4 to confirm that a full fade
  still takes 1 s and 5 s.

**Checkpoint**: US1 and US2 both pass.

---

## Phase 5: User Story 3 - Lamp only reacts when it is dark (Priority: P2)

**Goal**: once the time is known, motion only counts between sunset and sunrise. While the time
is unknown, motion counts day and night.

**Independent Test**:
- In daytime, wave. The strip stays off, and Daylight is on.
- Set `longitude` about 180° away from the real value, upload, and wave. The lamp works as in
  US1.
- Restore the real value afterwards.

### Implementation for User Story 3

- [X] T031 [US3] Add `time:` to section 3 of `esphome/buitenlamp.yaml` with two sources (research
  R8).
  - `- platform: sntp` with `id: sntp_time` and `servers:` `${ntp_server_1}` and
    `${ntp_server_2}`.
  - `- platform: homeassistant` with `id: ha_time`.
  - Each has an `on_time_sync:` that logs `t=%lu time synced (sntp)` or
    `t=%lu time synced (home assistant)`.
  - Comment: both set the one system clock, so whichever answers first makes the time known.
    Each re-syncs every 15 min while it can be reached.
  - Comment `# FR-020`.
- [X] T032 [US3] Add `interval:` to section 5 of `esphome/buitenlamp.yaml`.
  - `- interval: ${time_retry_interval}`.
  - `then:` an `if:` with `condition:` `not:` `time.has_time:` `id: sntp_time`.
  - Inside the `if:`: `component.update: sntp_time`, `component.update: ha_time`, and log
    `t=%lu time unknown, retrying`.
  - Comment `# FR-020`.
- [X] T033 [US3] Add `sun:` to section 3 of `esphome/buitenlamp.yaml`, with
  `latitude: ${latitude}`, `longitude: ${longitude}` and `time_id: sntp_time`.
  - Comment: darkness comes from the sun's elevation, not the local clock, so summer and winter
    time cannot shift it.
  - Comment `# FR-019, FR-021`.
- [X] T034 [US3] Start the `sensor:` list in section 6 of `esphome/buitenlamp.yaml` with
  `sun_elevation`, under `# Internal inputs`.
  - `platform: sun`, `type: elevation`, `id: sun_elevation`, `name: "Sun elevation"`,
    `internal: true`, `update_interval: ${daylight_check_interval}`.
  - Comment: the value is NaN until the time is known (research R7).
  - Comment `# FR-019`.
- [X] T035 [US3] Add `daylight` to the section-6 `binary_sensor:` list in
  `esphome/buitenlamp.yaml`, under `# Home Assistant entities` (data-model.md, Daylight).
  - `platform: template`, `id: daylight`, `name: "Daylight"`, `device_class: light`.
  - The lambda returns `{}` (no state) while `std::isnan(id(sun_elevation).state)`. Otherwise it
    returns `id(sun_elevation).state >= ${sun_horizon_deg}`.
  - Comment: no state means the time is unknown, and the logic treats that like dark (FR-023).
  - `on_press:` runs `- script.execute: dark_period_ended` (T036).
  - Comment `# FR-018, FR-019, FR-023, FR-025`.
- [X] T036 [US3] Add the darkness rule to the script `motion_started` in
  `esphome/buitenlamp.yaml` (data-model.md, When motion counts).
  - Wrap the counted branch in an `if:` with `condition:` `not:`
    `binary_sensor.is_on: daylight`. "Daylight not ON" means dark, or the time is unknown.
  - In the `else:` branch, log `t=%lu motion ON, ignored (daylight)` and leave `motion_counted`
    false.
  - Comment: a motion that starts in daylight never counts, even if it lasts into the dark
    (research R11). Motion after sunrise therefore cannot extend a running on period.
  - Add FR-018 and FR-023 to the script's comment.
  - Add the script `dark_period_ended` to section 5 (data-model T13), with comment
    `# FR-018, FR-023`.
    - `if:` `lambda: return id(motion_counted);`
    - `then:` set `motion_counted` to `'false'`, log
      `t=%lu dark period ended, motion no longer counted`, and run
      `script.execute: on_period_timer`.
    - Comment: a motion still going at sunrise ends there, so the on period runs from sunrise
      and the light fades out normally even if the person keeps moving (FR-018). If `on_press`
      also fires when the time first becomes known in daytime, the same applies, which FR-023
      allows.
- [ ] T037 [US3] Run the compile gate on `esphome/buitenlamp.yaml` and upload over WiFi.
  - Status 2026-11-01: compile gate passed (DEBUG, no warnings); upload still to do.

### Validation for User Story 3

- [ ] T038 [US3] Hands-on: run V8, V9, V11 (together with V19), V23 and V27 from
  `specs/001-pir-ledstrip-fade/quickstart.md`. After V9 and after V27, set `longitude` back to
  the real value and upload again.
- [ ] T039 [P] [US3] Hands-on: start V10. On 3 different days, compare the Daylight changes in the
  Home Assistant history with the published sunset and sunrise times.
  - This runs alongside Phases 6–8.
  - Record the result in section 9 of `specs/001-pir-ledstrip-fade/quickstart.md`.

**Checkpoint**: US1–US3 pass. The lamp can go outdoors: it reacts only after dark.

---

## Phase 6: User Story 4 - Home automation sees and controls the lamp (Priority: P3)

**Goal**: Home Assistant shows the lamp as one device with Buitenlamp, Motion, Daylight,
Brightness and Mode.
- "On" from Home Assistant means manual mode until "off".
- After 10 min without Home Assistant, the lamp goes back to automatic mode.

**Independent Test**:
- The device page shows the five entities.
- Walk past: Motion and Buitenlamp update within 2 s.
- Switch on in daylight: the light fades in and stays on, and Mode = manual.
- Switch off: the light fades out, and Mode = automatic.

### Implementation for User Story 4

- [X] T040 [US4] Add the globals `manual_mode` (FR-027, FR-029) and `need_fresh_motion` (FR-027)
  to `globals:` in `esphome/buitenlamp.yaml`.
  - Both are `type: bool`, `restore_value: no`, `initial_value: 'false'`.
  - Comment on `need_fresh_motion`: set by "off" while motion is active, and cleared when that
    motion ends (data-model.md, "Why `need_fresh_motion` exists").
- [X] T041 [US4] Add the Mode entity to `esphome/buitenlamp.yaml`.
  - Add a `text_sensor:` list to section 6 with `- platform: template`, `id: mode_text`,
    `name: "Mode"` and `update_interval: never`. It has no lambda.
  - Comment: scripts publish Mode only when it changes, so no text is built over and over
    (research R9, constitution III.1).
  - Comment `# FR-025`.
  - In `esphome: on_boot:`, before the stabilization delay, add `text_sensor.template.publish:`
    with `id: mode_text` and `state: "automatic"`.
- [X] T042 [US4] Add `brightness` to the section-6 `sensor:` list in `esphome/buitenlamp.yaml`,
  under `# Home Assistant entities`.
  - `platform: template`, `id: brightness`, `name: "Brightness"`, `unit_of_measurement: "%"`,
    `accuracy_decimals: 0`, `state_class: measurement`,
    `update_interval: ${brightness_report_interval}`.
  - `lambda: return roundf(${strip_level} * 100.0f);`
  - `filters:` with `- delta: 0.5`. Comment: values are whole percents, so any change gets
    through and an unchanged value is not sent.
  - Comment `# FR-025`.
- [X] T043 [US4] Add the scripts `manual_on` and `manual_off` to `esphome/buitenlamp.yaml`
  (data-model T7 and T9). Both have the comment `# FR-027`.
  - `manual_on`:
    - if `manual_mode` is false, set it to `'true'`, publish `"manual"` to `mode_text`, and log
      `t=%lu mode manual`;
    - then always `script.execute: lamp_fade_on`.
    - An "on" while the light is already on from motion gives no visible change, but the mode
      still becomes manual.
  - `manual_off`, in this order:
    - set `need_fresh_motion` to `!lambda 'return id(motion).state;'`;
    - set `motion_counted` to `'false'`;
    - `script.stop: on_period_timer`;
    - if `manual_mode` is true, set it to `'false'`, publish `"automatic"` and log
      `t=%lu mode automatic`;
    - log `t=%lu off from Home Assistant`;
    - `script.execute: lamp_fade_off`.
- [X] T044 [US4] Add a `switch:` list to section 6 of `esphome/buitenlamp.yaml` (research R9,
  contracts/ha-api.md).
  - `- platform: template`, `id: lamp_switch`, `icon: mdi:lightbulb`.
  - `name: None`, so it takes the device name Buitenlamp and the object id `buitenlamp`.
  - `restore_mode: DISABLED`. Comment: any other mode makes the switch act on a restored state at
    boot and fire `turn_off_action`.
  - `lambda: return id(strip).remote_values.is_on();`, so the switch is on while the light fades
    in or is on.
  - `turn_on_action:` runs `- script.execute: manual_on`, and `turn_off_action:` runs
    `- script.execute: manual_off`.
  - Comment: this is a switch, not the light, so every command runs even when the state does not
    change, and Home Assistant commands are told apart from motion.
  - Comment `# FR-024, FR-025, FR-027`.
  - *Superseded on 2026-11-02 by T067–T069 (Phase 9): the strip light replaces this switch.*
- [X] T045 [US4] Add manual mode and fresh motion to the script `motion_started` in
  `esphome/buitenlamp.yaml` (data-model.md, Details per script).
  - The "motion counts" condition becomes an `and:` of `not: binary_sensor.is_on: daylight` and
    `lambda: return !id(need_fresh_motion);`.
  - The `else:` log line gives the reason:
    `id(need_fresh_motion) ? "fresh motion needed" : "daylight"`.
  - Add `lambda: return !id(manual_mode);` to the condition that runs `lamp_fade_on`.
    - In ManualOn the motion is still recorded: `motion_counted` is set and `on_period_timer` is
      stopped.
    - The light is not touched.
  - The script's comment becomes `# FR-002, FR-004, FR-006, FR-018, FR-023, FR-027`.
- [X] T046 [US4] Update the scripts `motion_ended` and `on_period_timer` in
  `esphome/buitenlamp.yaml`.
  - `motion_ended`: its first action always sets `need_fresh_motion` to `'false'`. Add FR-027 to
    its comment.
  - `on_period_timer`: run `lamp_fade_off` only `if:` `lambda: return !id(manual_mode);`.
    - Comment: expiring in ManualOn changes nothing.
    - A running timer only tells `fallback_timer` that there was recent motion.
- [X] T047 [US4] Add the script `fallback_timer` to `esphome/buitenlamp.yaml` (data-model T10,
  comment `# FR-029`).
  - `mode: restart`, then `delay: ${manual_fallback_ms}ms`.
  - Then an `if:` whose condition is an `and:` of `lambda: return id(manual_mode);` and `not:`
    `api.connected:` with `state_subscription_only: true`. Its `then:`:
    - set `manual_mode` to `'false'`, publish `"automatic"`, and log
      `t=%lu Home Assistant unreachable, mode automatic`;
    - an `if:` whose condition is an `and:` of `not: binary_sensor.is_on: daylight` and an `or:`
      of `lambda: return id(motion_counted);` and `script.is_running: on_period_timer`;
    - `then:` log `t=%lu stays on (recent motion in the dark)`;
    - `else:` `script.execute: lamp_fade_off`.
  - Comment: Home Assistant is checked here, when the timer expires, because
    `on_client_connected` fires before Home Assistant subscribes to states (research R10), so a
    reconnect cannot stop this timer. Every disconnect restarts it, so the lamp falls back only
    after `${manual_fallback_ms}` without Home Assistant.
- [X] T048 [US4] Add the client triggers to `api:` in section 2 of `esphome/buitenlamp.yaml`
  (research R10, data-model T11, comment `# FR-029`).
  - `on_client_connected:` only logs `t=%lu API client connected`.
    - Comment: this trigger fires before the client subscribes to states, so the condition
      below is still false here (research R10). `fallback_timer` checks for Home Assistant
      itself when it expires (T047).
  - `on_client_disconnected:` logs `t=%lu API client disconnected`. Then, if `not:`
    `api.connected:` with `state_subscription_only: true`, it runs
    `script.execute: fallback_timer`.
    - Comment: this trigger fires after the client has been removed, so the condition sees
      only the clients that are left (research R10).
  - Comment: `esphome logs` also subscribes to states, so use `--no-states` when testing the
    fallback.
- [ ] T049 [US4] Run the compile gate on `esphome/buitenlamp.yaml` and upload over WiFi.
  - Status 2026-11-01: compile gate passed (DEBUG, no warnings); upload still to do.

### Validation for User Story 4

- [ ] T050 [US4] Hands-on: run V13, V14, V15, V16 and V20 from
  `specs/001-pir-ledstrip-fade/quickstart.md`.
  - For V13, use the discovery time noted in T013.
  - Check that the object ids match `specs/001-pir-ledstrip-fade/contracts/ha-api.md`:
    `buitenlamp`, `motion`, `daylight`, `brightness` and `mode`.
- [ ] T051 [US4] Hands-on: run V17 and V18 from `specs/001-pir-ledstrip-fade/quickstart.md`.
  Watch the log with `--no-states`.
  - V17: 1 h at night with the Home Assistant entry disabled.
  - V18: the fallback, once without and once with a 5-minute interruption.

**Checkpoint**: US1–US4 pass.

---

## Phase 7: User Story 5 - Owner adjusts the timing (Priority: P4)

**Goal**: the owner changes a timing value in one labelled place and uploads over WiFi. The lamp
uses the new value. An out-of-range value stops the build.

**Independent Test**: set `on_period_ms` to `"120000"`, upload over WiFi and trigger once. The
light stays on for 120–121 s. Restore the value afterwards.

### Implementation for User Story 5

- [X] T052 [US5] Make the first action of `esphome: on_boot:` in `esphome/buitenlamp.yaml` a
  `lambda:` that holds only `static_assert`s, one or more for each numeric owner setting
  (contracts/settings.md, comment `# FR-013`). The ranges, verbatim from the contract:
  - `fade_in_ms` 100–10000; `on_period_ms` 1000–86400000; `fade_out_ms` 100–60000;
  - `pir_stabilization_ms` 0–120000; `pir_min_pulse_ms` 1–1000;
    `manual_fallback_ms` 60000–86400000;
  - `num_leds` 1–240;
  - `on_red`, `on_green`, `on_blue` and `on_white` each 0.0–1.0, and their sum > 0.0 ("not all
    0").
  - Each message names the setting, its range and its unit, for example
    `"num_leds must be 1-240 LEDs"`.
  - Comment: ESPHome itself checks the pins (including one pin used twice), the channel order,
    the coordinates and the log level.
- [X] T053 [US5] Check the `substitutions:` block of `esphome/buitenlamp.yaml` against
  contracts/settings.md and US5 scenario 2.
  - Every owner setting says what it controls, its unit and its range.
  - Search the rest of the file for numbers in automations, scripts, filters and lambdas. Move
    any tunable into a substitution (constitution V.1).
- [ ] T054 [US5] Prove the range check works (section 4 of
  `specs/001-pir-ledstrip-fade/quickstart.md`).
  - Set `num_leds` to `"300"` in `esphome/buitenlamp.yaml`. `esphome compile` must stop with the
    `num_leds` `static_assert` message.
  - Set it back, then confirm the compile gate passes and upload over WiFi.
  - Status 2026-11-01: proven with `esphome -s num_leds 300 compile …` (the file was not
    edited): the build stops at `buitenlamp.yaml:93` with "num_leds must be 1-240 LEDs". The
    compile gate with the file's values passes at DEBUG and INFO; upload still to do.

### Validation for User Story 5

- [ ] T055 [US5] Hands-on: run V21 and V24 from `specs/001-pir-ledstrip-fade/quickstart.md`.
  - V21: an on period of 120 s uploaded over WiFi, with the edit and upload done in under 5 min.
    Restore the default afterwards.
  - V24: an upload with a wrong key is refused. A normal upload restarts the lamp off and in
    automatic mode.

**Checkpoint**: all five user stories pass.

---

## Phase 8: Polish & Cross-Cutting Concerns

**Purpose**: the review, traceability and resource gates, and delivery.

- [ ] T056 Review gate: check `esphome/buitenlamp.yaml` against constitution v2.0.0.
  - Every script and automation has a `# FR-…` comment that matches data-model.md (IV.3).
  - Only `lamp_fade_on` and `lamp_fade_off` call the light, and each lambda is one decision or
    one calculation (IV.2).
  - `current_values` appears only in the `strip_level` substitution (IV.2).
  - Run V26 from `specs/001-pir-ledstrip-fade/quickstart.md` and record the result (FR-022).
  - `millis()` appears only in log lines (II.2).
  - There is no `restore_value: yes`, `web_server`, `captive_portal`, `ap:` or `effects:` (III.3,
    V.5).
  - Pins appear only in `pir_raw` and `strip`, through substitutions (IV.1).
  - The pin map in `specs/001-pir-ledstrip-fade/contracts/hardware.md` names the same GPIOs as
    `pin_pir` and `pin_led_data` (GPIO4 and GPIO5 by default), and the wiring built in T006
    matches it (V.3, gate 4).
  - Every non-obvious setting has a comment saying why.
  - Record any deviation in the Complexity Tracking table of
    `specs/001-pir-ledstrip-fade/plan.md`.
  - Status 2026-11-01: every check above passes, V26 included (recorded in quickstart section
    9), with no new deviations. Still to do: confirm the wiring built in T006 matches the pin
    map.
- [X] T057 [P] Traceability gate: in section 10 of `specs/001-pir-ledstrip-fade/quickstart.md`,
  update the "Implemented by" column to the ids actually used. Add `fade_length_ms` to FR-006.
- [X] T058 [P] Bring the design docs in line with what was built.
  - Add `fade_length_ms` to the Globals table in `specs/001-pir-ledstrip-fade/data-model.md`.
  - In the Source Code tree of `specs/001-pir-ledstrip-fade/plan.md`, note that `on_boot` sits in
    `esphome:`, and that `binary_sensor:` and `sensor:` each appear once, in section 6.
- [ ] T059 Prepare the delivered firmware. Do this after T057, which edits the same quickstart
  file.
  - Set `log_level` back to `"INFO"` in `esphome/buitenlamp.yaml`.
  - Run the compile gate. It must show no warnings that point to the file.
  - Write into section 9 of `specs/001-pir-ledstrip-fade/quickstart.md`:
    - RAM and flash use, in % and bytes (limits 75 % and 90 %);
    - the ESPHome version, framework and variant.
  - Upload over WiFi.
  - Status 2026-11-01: `log_level` is `"INFO"`, the INFO compile gate passes with no warnings,
    and section 9 has the RAM, flash and version rows. Still to do: the upload. If T010 sets
    `DEBUG` for validation, set `INFO` again and recompile before this upload.
- [ ] T060 Hands-on: smoke-test the INFO firmware with V1, V3 and V4 timed by stopwatch, and one
  on/off from Home Assistant. Do this after dark, because the darkness rule is now active. At
  INFO there are no `t=` lines, so time the on period from when Motion turns off in Home
  Assistant, allowing for its delay of up to 2 s. Then fill in the remaining rows of section 9 of
  `specs/001-pir-ledstrip-fade/quickstart.md`:
  - strip current at full brightness, with `num_leds`;
  - scenarios passed, listing any exceptions;
  - the date and `project: version`.
- [ ] T061 Hands-on: start V22 on the day of delivery, and write the start date in section 9 of
  `specs/001-pir-ledstrip-fade/quickstart.md`. After 60 days, repeat V3 and V4 and record the
  result (plan.md, Complexity Tracking).

---

## Phase 9: Change 2026-11-02 - Brightness and colour from Home Assistant (extends User Story 4)

**Goal**: Home Assistant shows Buitenlamp as a light with on/off, brightness and colour. The
chosen brightness and colour are used for every "on", from motion too, and are kept after a
power cut (FR-017, FR-024, FR-025, FR-027, FR-031; research R18).

**Independent Test**: V28 in quickstart.md: choose 30 % red in Home Assistant, switch off, wave
after dark, power-cycle and wave again. Each "on" is 30 % red, and a full fade-in still takes
0.9–1.1 s.

### Implementation for the change

- [X] T062 Update the design documents for the change.
  - spec.md: Session 2026-11-02, User Story 4 (scenarios 9 and 10), edge cases, FR-013, FR-017,
    FR-024, FR-027, the new FR-031 and SC-016.
  - plan.md (summary, storage, Constitution Check), research.md (R6, R9 note, the new R18),
    data-model.md (globals, entities, fade lengths, T7, T9, T12).
  - contracts: ha-api.md (light entity, version 1.1.0), settings.md (`on_brightness`),
    hardware.md (power for all four channels).
  - quickstart.md: V13, V14, the new V28, trials and traceability.
- [X] T063 In section 1 of `esphome/buitenlamp.yaml`, add the owner setting
  `on_brightness: "1.0"` (default brightness; fraction of full; 0.01–1.0; FR-017).
  - Change the on-colour comment: the `on_*` values are defaults, used until a brightness and
    colour are chosen in Home Assistant (FR-031).
  - Add `static_assert`s for `on_brightness` (0.01–1.0) to the `on_boot` range-check lambda.
  - Set `project: version` to `"1.1.0"` (contracts/ha-api.md, Compatibility rules).
- [X] T064 Add the globals to section 5 (data-model.md, Globals).
  - `chosen_brightness`, `chosen_color_brightness`, `chosen_red`, `chosen_green`, `chosen_blue`
    and `chosen_white`: `type: float`, `restore_value: yes`. Initial values `${on_brightness}`,
    `'1.0'`, `${on_red}`, `${on_green}`, `${on_blue}` and `${on_white}`.
  - `lamp_call_active`: `type: bool`, `restore_value: no`, `initial_value: 'true'`. Comment:
    true while a fade script calls the light, and at boot until `on_boot` clears it.
  - Section comment: the lamp's state is never saved; only the on-setting is, and only when it
    changes (FR-009, FR-030, FR-031).
- [X] T065 Change `lamp_fade_on` (data-model.md, Fade lengths).
  - `fade_length_ms` = `${fade_in_ms} × min(1, |B − ${strip_level}| / B)`, with B =
    `chosen_brightness`.
  - `light.turn_on` takes `brightness`, `color_brightness`, `red`, `green`, `blue` and `white`
    from the `chosen_*` globals.
  - Set `lamp_call_active` to true straight before `light.turn_on` and to false straight after.
- [X] T066 Change `lamp_fade_off` in the same way.
  - `fade_length_ms` = `${fade_out_ms} × min(1, ${strip_level} / B)`.
  - Use `light.control` with `state: false` and the `chosen_*` values, so the strip's target
    values keep the on-setting for a plain "on" from Home Assistant.
  - Set and clear `lamp_call_active` around the call. Add FR-031 to the comment.
- [X] T067 Expose the strip light in section 4.
  - Remove `internal: true`; set `name: None`, so it takes the device name Buitenlamp and the
    object id `buitenlamp`.
  - `default_transition_length: ${fade_in_ms}ms`, so a colour change from Home Assistant fades.
  - `on_state:` runs `script.execute: ha_command` only `if:` `lambda: return
    !id(lamp_call_active);`. Comment: every light call runs `on_state` synchronously, so with
    the flag clear the call came from Home Assistant (research R18).
  - Comment `# FR-024, FR-025, FR-027`.
- [X] T068 Add the script `ha_command` to section 5 (data-model T7, T9; comment
  `# FR-027, FR-031`).
  - If `id(strip).remote_values.is_on()`: one `lambda` copies the strip's target brightness,
    colour brightness, red, green, blue and white into the `chosen_*` globals; log
    `t=%lu on from Home Assistant, brightness %.2f`; `script.execute: manual_on`.
  - Else: `script.execute: manual_off`.
- [X] T069 Remove the `switch:` section (`lamp_switch`) and update the header comment's section
  list. Move the "off from Home Assistant" log line in `manual_off` if needed, so each command
  logs once.
- [X] T070 In `esphome: on_boot:`, straight after the range checks, run
  `script.execute: lamp_fade_off`. Comment: loads the stored on-setting into the strip's target
  values and clears `lamp_call_active`, so a plain "on" from Home Assistant uses it (FR-031,
  research R18).
- [ ] T071 Run the compile gate at DEBUG and at INFO, and record RAM and flash in section 9 of
  `specs/001-pir-ledstrip-fade/quickstart.md`. Then build and upload with `esphome run`
  (Hands-on).
  - Status 2026-11-02: compile gate passed at DEBUG and INFO with no warnings (RAM 34.2 %,
    flash 47.0 % at INFO, `num_leds` 26), recorded in section 9. Upload still to do.
- [X] T072 [P] Update `README.md`: the light entity instead of the switch, how to choose
  brightness and colour, and the power supply size for all four channels.

### Validation for the change

- [ ] T073 Hands-on: in Home Assistant, delete the old `switch.buitenlamp` (now unavailable) on
  the device page, and change any automations or dashboards to `light.buitenlamp`.
- [ ] T074 Hands-on: run V28 (3 times), then repeat V3, V4, V13, V14, V16 and V20 with the
  light entity. Check that the 5 V supply is sized for all four channels (contracts/hardware.md).

**Checkpoint**: brightness and colour come from Home Assistant, survive a power cut, and the
fade timings of US1 and US2 still hold.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: no dependencies. T004 needs T002. T005 needs T001 and T003.
- **Foundational (Phase 2)**: needs T004 and T005 to compile. T012 needs the board, not the
  wiring.
- **US1 (Phase 3)**: needs Foundational. Its validation (T024, T025) also needs the hardware from
  T006.
- **US2 (Phase 4)**: needs US1, because it refines US1's scripts.
- **US3 (Phase 5)**: needs US1. V23 (T038) and V10 (T039) also need the Home Assistant connection
  from T013.
- **US4 (Phase 6)**: needs US1 and US3, because T045 and T047 use `daylight`.
- **US5 (Phase 7)**: T052–T054 need only Foundational. T055 needs US4, because V24 checks the
  mode after the restart.
- **Polish (Phase 8)**: needs all stories.
- **Change 2026-11-02 (Phase 9)**: needs US4 (Phase 6) and T052 (the `on_boot` range checks).
  T063–T070 edit `esphome/buitenlamp.yaml` in order; T072 can run alongside. After Phase 9,
  T059–T061 (final firmware, smoke test, 60-day run) apply to version 1.1.0.

### Within Each User Story

1. Edit `esphome/buitenlamp.yaml` in task order. Each task builds on the one before.
2. Run the compile gate and upload over WiFi.
3. Run the story's hands-on scenarios.

### Parallel Opportunities

- Setup: T001, T002, T003 and T006 touch different files or only hardware.
- T006 (hardware) can run alongside all of Phase 2.
- T039 (V10, three days) runs in the background during Phases 6–8.
- T057 and T058 edit different documents.
- Every other task edits `esphome/buitenlamp.yaml` and must run in order. Two agents editing that
  file at the same time would conflict.

---

## Parallel Example: Setup and Polish

```text
# Setup, all at once:
Task: "T001 Create .gitignore at the repository root"
Task: "T002 Create requirements.txt with esphome==2026.10.1"
Task: "T003 Create esphome/secrets.example.yaml"
Hands-on: "T006 Build the bench hardware per contracts/hardware.md"

# While Phases 6-8 are built:
Hands-on: "T039 V10: compare Daylight with sunset and sunrise on 3 days"

# Polish:
Task: "T057 Update the traceability table in quickstart.md section 10"
Task: "T058 Add fade_length_ms to data-model.md; update the layout note in plan.md"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup.
2. Complete Phase 2: Foundational (CRITICAL: blocks all stories).
3. Complete Phase 3: User Story 1.
4. **STOP and VALIDATE**: T024 and T025. The result is a motion lamp with smooth fades that works
   without a network. It still reacts in daylight, so keep it on the bench.

### Incremental Delivery

1. Setup and Foundational: a board that updates over WiFi.
2. US1: motion fades on and off (MVP).
3. US2: stays on with presence, and recovers smoothly during a fade-out.
4. US3: reacts only in the dark. From here the lamp can be mounted outdoors.
5. US4: Home Assistant control, manual mode and the fallback.
6. US5: range checks for the settings, and timing changes over WiFi.
7. Polish: the gates, INFO firmware and the 60-day run.

Every phase ends with a compile, a WiFi upload and its scenarios, so the device always runs a
working lamp.

---

## Notes

- Only the first load goes over USB (T012). Every later upload goes over WiFi.
- Keep `log_level` at `DEBUG` until T059. The timing scenarios read the `t=` timestamps from the
  log.
- Always watch the log with `--no-states`. Otherwise the log viewer counts as a Home Assistant
  client and holds off the manual-mode fallback.
- Mark each task `[X]` when it is done. Commit after each phase.
