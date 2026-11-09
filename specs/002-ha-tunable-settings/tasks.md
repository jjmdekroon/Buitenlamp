---

description: "Task list for lamp settings adjustable from Home Assistant (ESPHome, 1.2.0)"
---

# Tasks: Lamp Settings Adjustable from Home Assistant

**Input**: Design documents from `/specs/002-ha-tunable-settings/`

**Prerequisites**: plan.md, spec.md, research.md (S1–S11), data-model.md, contracts/ (ha-api.md,
settings.md), quickstart.md (W1–W22)

**Tests**: there are no automated tests, because constitution v2.0.0 drops host-side tests.
Each story ends with two checks instead:
- the compile gate (`esphome config` and `esphome compile`);
- the on-device scenarios from quickstart.md (W1–W22) that prove the story.

**Organization**: tasks are grouped by user story. Almost every task edits the same file,
`esphome/buitenlamp.yaml`, so those tasks run in order. Most of the parallel work is hands-on
validation and documentation.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: can run in parallel (different files, no dependencies)
- **[Story]**: the user story from spec.md (US1–US3)
- **Hands-on**: needs a person with the hardware, Home Assistant or private values. An agent can
  prepare these tasks and explain the steps, but cannot do them.

## Conventions (apply to every task)

**Paths** are from the repository root. Commands run in PowerShell from the repository root and
use `.venv\Scripts\esphome`.

**Host name**: the device is `kantoorlamp` (commit e1dc5dc), so its address is
`kantoorlamp.local`. Some documents still say `buitenlamp.local`: T034 fixes the README, T035
the 001 quickstart.

**Layout of `esphome/buitenlamp.yaml`**: as in 001, with one new key in section 2 and three new
keys at the end of section 6:

1. `substitutions:`: owner settings, then fixed constants
2. `esphome:` (including `on_boot:`), `esp32:`, **`preferences:`**, `logger:`, `wifi:`, `api:`,
   `ota:`
3. `time:`, `sun:`
4. `light:` (the strip)
5. `globals:`, `script:`, `interval:`
6. `binary_sensor:`, `sensor:`, `text_sensor:`, then **`number:`, `datetime:`, `select:`**

- ESPHome rejects a top-level key that appears twice. Each new key therefore appears once.
- The three new keys of section 6 sit under one comment line: `# Settings chosen in Home
  Assistant`.
- `in_window` and `motion_allowed` go in the existing `binary_sensor:` list.
  - `motion_allowed` goes under `# Internal inputs`, after `pir_raw`.
  - `in_window` goes under `# Home Assistant entities`, after `daylight`.

**Requirement labels** (constitution IV.3, plan.md IV.3):
- The existing labels in the file are 001's and stay as they are, without a prefix.
- This feature's ids get the prefix `002/`, for example `# FR-003, FR-005, 002/FR-007`.
- A changed script keeps its 001 ids and gets this feature's ids next to them.
- Each task gives the ids to add.

**Numbers** (constitution V.1):
- Every tunable, range, step, interval and option index comes from a substitution.
- 001 allows only a few literals: unit conversions (`100.0f` for percent), the full level `1.0f`
  and `0` for "no time left". This feature adds two unit conversions:
  - `1000.0f` (seconds to ms), only in the three `*_now_ms` substitutions;
  - `60` (minutes per hour), only in the `in_window` lambda.

**Contract names** (contracts/ha-api.md, research S3, S8):
- Spell the seven entity names exactly as given. The lamp derives the storage key from the
  name, so a renamed entity loses its stored value: "On period", "Fade-in time",
  "Fade-out time", "Time window start", "Time window end", "Reaction mode" and
  "In time window".
- Spell the four Reaction mode options exactly, in this order: `only when dark`,
  `only in time window`, `in time window and dark`, `always`. The lamp stores the option's
  index, so the order may never change; new options may only be appended.

**Lambdas** (constitution II.1, III.1, IV.2):
- `in_window` and `motion_allowed` run on every main-loop pass. They must not build or compare
  strings and must not allocate.
- Do not give a local variable the same name as an entity id: `id(in_window)` expands to the
  variable `in_window`, so a local `in_window` would hide it.

**Log lines**: as in 001 (research R17).
- Use `logger.log` with `level: DEBUG` and `tag: lamp`.
- Start every message with `t=%lu`, and pass `(unsigned long) millis()` as the first argument.
- Print `uint32_t` values with `%lu` and an `(unsigned long)` cast. `%u` gives a format warning
  on ESP-IDF 5, which fails the compile gate.

**Compile gate and upload**: the last implementation task of every story runs these commands:

```powershell
.venv\Scripts\esphome config esphome/buitenlamp.yaml
.venv\Scripts\esphome compile esphome/buitenlamp.yaml
.venv\Scripts\esphome upload esphome/buitenlamp.yaml --device kantoorlamp.local
.venv\Scripts\esphome logs esphome/buitenlamp.yaml --device kantoorlamp.local --no-states
```

- `esphome config` must end with "Configuration is valid!".
- `esphome compile` must show no warnings that point to `esphome/buitenlamp.yaml`.
- The upload needs the lamp on the network. If it cannot be reached, record the compile result
  and leave the upload as a status note on the task.
- Record each scenario's result in section 5 of `specs/002-ha-tunable-settings/quickstart.md`
  (the "Scenarios W1–W22 passed" row). Record failed trials in section 4.
- Run each scenario as many times as the Trials table in section 3 of quickstart.md says.

**Power-cut trials** (002/FR-004, SC-004, research S8):
- Cut the power **2 to 5 seconds** after the last change. Do not wait longer: the trial must
  show that a change is safe 2 s after it was made.
- At `DEBUG`, the log line `[preferences] Writing N items: … written` shows the moment a change
  reached flash. It must appear within 0.5 s of a setting change and within 1.5 s of a
  brightness or colour change.
- One `Writing` line about 60 s after every boot is ESPHome's safe-mode boot counter, not a
  setting (research S8).

**If `esphome config` rejects a key or a C++ call named in a task**:
- look up the current name in the ESPHome 2026.10.1 sources in
  `.venv/Lib/site-packages/esphome/components/`;
- use that name;
- note the change in `specs/002-ha-tunable-settings/research.md`.

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: a feature branch, a known-good starting build and a lamp that runs 1.1.0 in Home
Assistant.

- [X] T001 Create the branch `002-ha-tunable-settings` from `main` and switch to it
  (plan.md names this branch).
  - Run `git checkout -b 002-ha-tunable-settings`.
  - The untracked folder `specs/002-ha-tunable-settings/` comes along. Commit it as the first
    commit on the branch only if the owner asks.
- [X] T002 [P] Run the compile gate on the unchanged `esphome/buitenlamp.yaml` (1.1.0) as a
  baseline.
  - Run `esphome config` and `esphome compile` only; do not upload.
  - Check that `.venv\Scripts\esphome version` prints `Version: 2026.10.1`.
  - The build must pass with no warnings, so later failures belong to this feature.
  - Note RAM and flash for comparison. 001 recorded 34.2 % RAM and 47.0 % flash at INFO.
  - **Status (2026-11-09)**: passed, no warnings; RAM 34.2 % (109758 bytes), flash 47.0 %
    (862228 bytes), the same as 001.
    - Built from a copy of the file next to dummy secrets: `esphome config` on the real
      `esphome/secrets.yaml` fails with "Invalid key format, please check it's using base64" for
      `api_encryption_key`.
    - The compile gates of this feature are built the same way. Uploading needs the real key,
      so 001 T005 (WiFi details and key) must be redone first.
- [ ] T003 [P] Hands-on: make sure the lamp runs 1.1.0 and is known to Home Assistant.
  - The 002 scenarios need four 001 tasks done: T005 (WiFi details), T010 (location and strip
    length), T012 (first upload) and T013 (Home Assistant).
  - See `specs/001-pir-ledstrip-fade/tasks.md` for those tasks.
  - Do step 1 of section 1 in `specs/002-ha-tunable-settings/quickstart.md` (001 quickstart
    sections 1–5), but leave `log_level` as it is; T005 sets it. Steps 2–4 need the 1.2.0
    file: T020 sets the time zone, and the compile gate tasks (T011, T018, T029) build, check
    the range and upload.
  - Note Home Assistant's time zone (Settings → System → General) for T020.

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: the file announces feature 002, version 1.2.0 and the label convention, logs at
DEBUG for validation, and stores every change within 2 s, which every story's power-cut check
needs.

**⚠️ CRITICAL**: no user story work can begin until this phase is complete.

- [X] T004 Update the header comment of `esphome/buitenlamp.yaml`.
  - In the section list:
    - section 2 becomes `esphome (including on_boot), esp32, preferences, logger, wifi, api,
      ota`;
    - section 6 becomes `binary_sensor, sensor, text_sensor, number, datetime, select`,
      described as "inputs, Home Assistant entities and settings".
  - The spec line names both folders: `specs/001-pir-ledstrip-fade/` and
    `specs/002-ha-tunable-settings/`.
  - Add one line on the labels: `FR-0xx` without a prefix is from 001; `002/FR-0xx` is from 002.
- [X] T005 In section 2 of `esphome/buitenlamp.yaml`, set the version and the log level.
  - Set `esphome: project: version:` to `"1.2.0"`.
  - Replace the 1.1.0 comment above it with: "1.2.0: the on period, fade times, time window
    and reaction mode can be chosen in Home Assistant; seven new entities, nothing renamed
    (specs/002-ha-tunable-settings/contracts/ha-api.md)".
  - Set `log_level` in `substitutions:` to `"DEBUG"`, as section 1 of
    `specs/002-ha-tunable-settings/quickstart.md` requires for validation. T041 sets it back.
- [X] T006 Store every change within 2 s: add `flash_write_interval` and a `preferences:` block
  to `esphome/buitenlamp.yaml` (contracts/settings.md, research S8, data-model.md "When a value
  is stored").
  - Fixed constants in `substitutions:`, after `time_retry_interval`: add
    `flash_write_interval: "500ms"`.
    - Comment: "how often ESPHome writes changed values to flash. 500 ms stores every change
      within the 2 s of 002/FR-004: a setting within 0.5 s, brightness and colour (checked once
      a second) within 1.5 s. A longer value breaks 002/FR-004 (research S8)".
  - Section 2, after `esp32:` and before `logger:`, add:

    ```yaml
    preferences:
      flash_write_interval: ${flash_write_interval}
    ```

    - Label `# 002/FR-004, FR-031`.
    - Comments:
      - ESPHome's default is 60 s, so a power cut within a minute of a change would lose it;
      - flash is still written only for values that changed, and with nothing to write the sync
        returns at once (constitution III.3);
      - at DEBUG, every write logs `[preferences] Writing N items: … written`.
  - In section 5, in the comment above `chosen_brightness` (the one that ends "saved in flash
    only when it changes (FR-017, FR-031)"), add one line: "ESPHome checks them once a second
    and writes a change within 1.5 s (`preferences:`, 002/FR-004)."
  - Do not add `update_interval` to the `chosen_*` globals: the 1.5 s relies on ESPHome's
    default of 1 s.
  - Do not add any lambda that calls `global_preferences->sync()` (research S8, rejected
    alternative).
  - Run `esphome config`. It must end with "Configuration is valid!".

**Checkpoint**: foundation ready. The user stories can start.

---

## Phase 3: User Story 1 - Owner changes how long the light stays on (Priority: P1) 🎯 MVP

**Goal**: the On period is a setting on the lamp's device page in Home Assistant, 1–86400 s,
default 60 s. The next on period uses it, and it is kept after a power cut or update.

**Independent Test**: in Home Assistant, set On period to 120 s. Wave once after dark and step
out of view. The log shows "on period ended" 120–121 s after `motion OFF`. Then set On period
to 90 s, cut the power 2–5 s later and restore it: the card still shows 90 s and the next
cycle lasts 90–91 s.

### Implementation for User Story 1

- [X] T007 [US1] Replace the on-period setting in `substitutions:` of `esphome/buitenlamp.yaml`
  (contracts/settings.md, research S1, S7).
  - Owner settings: replace `on_period_ms: "60000"` with `on_period_s: "60"`.
    - Comment: "default On period, used until it is chosen in Home Assistant; s; 1-86400
      (002/FR-001, 002/FR-005)".
    - Comment above the owner settings, added once: these timing values are now only the
      defaults of the Home Assistant settings. A 1.1.0 value in ms divided by 1000 gives the
      same value in s.
  - Fixed constants: add `on_period_min_s: "1"`, `on_period_max_s: "86400"` and
    `on_period_step_s: "1"`, with the comment "range and step of On period (002/FR-001)".
  - Below `strip_level`, add the shared expression
    `on_period_now_ms: "((uint32_t) lroundf(id(on_period).state * 1000.0f))"`.
    - Comment: "the current On period in ms. The one conversion, used by `on_period_timer`
      (constitution IV.2)".
- [X] T008 [US1] In the range-check lambda of `esphome: on_boot:` in `esphome/buitenlamp.yaml`,
  replace the `on_period_ms` check (research S1, S9).
  - New check: `static_assert(${on_period_s} >= ${on_period_min_s} && ${on_period_s} <=
    ${on_period_max_s}, "on_period_s must be ${on_period_min_s}-${on_period_max_s} s");`
  - Substitutions also work inside the message string, so the range is written only once, in
    `substitutions:`.
- [X] T009 [US1] Add the `number:` list to section 6 of `esphome/buitenlamp.yaml`, after
  `text_sensor:`, under the comment `# Settings chosen in Home Assistant`. Its first entity is
  On period (data-model.md, research S1, S8).
  - `platform: template`, `id: on_period`, `name: "On period"`.
  - `entity_category: config`, `mode: box`, `unit_of_measurement: s`.
  - `min_value: ${on_period_min_s}`, `max_value: ${on_period_max_s}`,
    `step: ${on_period_step_s}`, `initial_value: ${on_period_s}`.
  - `optimistic: true` and `restore_value: true`.
  - `on_value:` logs `t=%lu on period %.0f s` with `x`. It also fires for the restored value
    at boot, so the log shows which value the lamp uses (W6).
  - Comments:
    - `# 002/FR-001, 002/FR-002, 002/FR-004, 002/FR-005, 002/FR-007`;
    - ESPHome refuses a value outside min/max, logs a warning and keeps the old value
      (002/FR-002);
    - the value is stored only when it changes, within 0.5 s (`preferences:`, constitution
      III.3, 002/FR-004);
    - `mode: box`, because a slider cannot set an exact value between 1 and 86400;
    - the name is the storage key (contracts/ha-api.md).
- [X] T010 [US1] In the script `on_period_timer` in `esphome/buitenlamp.yaml`, replace
  `delay: ${on_period_ms}ms` with `delay: !lambda 'return ${on_period_now_ms};'`
  (data-model T4, research S7).
  - Add `002/FR-003, 002/FR-007` to the script's label.
  - Comment: the delay reads On period once, when it starts, so a running on period keeps its
    length and the next one uses the new value.
  - Search `esphome/buitenlamp.yaml` for `on_period_ms`. Nothing may remain.
- [X] T011 [US1] Run the compile gate on `esphome/buitenlamp.yaml` and upload over WiFi (see
  Conventions).
  - First prove the range check: `.venv\Scripts\esphome -s on_period_s 0 compile
    esphome/buitenlamp.yaml` must stop with the `on_period_s` `static_assert` message.
  - The file itself is not edited for this check.
  - **Status (2026-11-09)**: config valid; compile passed, no warnings; RAM 34.2 % (109862
    bytes), flash 47.7 % (874640 bytes). The range check stopped with "static assertion
    failed: on_period_s must be 1-86400 s". **Upload not done**: it needs the real API key in
    `esphome/secrets.yaml` (see T002); the upload of the full 1.2.0 after T029 covers it.

### Validation for User Story 1

- [ ] T012 [US1] Hands-on: run the On period parts of the scenarios in section 2 of
  `specs/002-ha-tunable-settings/quickstart.md`. Record the results in sections 4 and 5.
  - **W1, On period row**: right after the first upload, the Configuration section shows On
    period 60 s. Record its entity id.
  - **W2, On period**: `number.set_value` with 0.5 and with 90000 is refused each time.
  - **W3**: 10 s, 2 min and 10 min, 5 trials each (SC-002). Do 4 of the 10 SC-008 checks
    here: change On period and time how fast the card confirms it (within 2 s).
  - **W5, first half**: shorten On period while the light is on.
  - **Power cut, once**: set On period to 90 s; check that a `[preferences] Writing …`
    line appears within 0.5 s; cut the power 2–5 s after the change and restore it. The card
    shows 90 s and one wave lasts 90–91 s. The full W6 runs in T039.
  - Motion counts only after dark until US3 is built. To test during the day, use the
    `longitude` trick of 001 V9. If US3 is already built, choose Reaction mode `always`
    instead, and set it back afterwards.

**Checkpoint**: MVP. The owner sets the on period from a phone, without a PC or upload.

---

## Phase 4: User Story 2 - Owner changes the fade-in and fade-out times (Priority: P2)

**Goal**: Fade-in time (0.1–10 s, default 1.0 s) and Fade-out time (0.1–60 s, default 5.0 s)
are settings in Home Assistant. They are used by every following fade: from motion, from Home
Assistant on/off, and for Home Assistant's own brightness and colour changes.

**Independent Test**: set Fade-in time to 3 s and Fade-out time to 10 s. Wave after dark.
- The log shows "fade complete" 2.7–3.3 s after the counted `motion ON`.
- After the on period, the fade-out takes 9–11 s.
- Switching on and off from Home Assistant uses the same times.
- After a power cut 2–5 s after the last change, the card still shows 3.0 s and 10.0 s.

### Implementation for User Story 2

- [X] T013 [US2] Replace the fade settings in `substitutions:` of `esphome/buitenlamp.yaml`
  (contracts/settings.md, research S1, S7).
  - Owner settings:
    - replace `fade_in_ms: "1000"` with `fade_in_s: "1.0"`. Comment: "default Fade-in time,
      used until it is chosen in Home Assistant; s; 0.1-10 (002/FR-001, 002/FR-005)";
    - replace `fade_out_ms: "5000"` with `fade_out_s: "5.0"`. Comment: "default Fade-out
      time, used until it is chosen in Home Assistant; s; 0.1-60 (002/FR-001, 002/FR-005)";
    - if T007 has not added the "now only the defaults" comment above the owner settings, add
      it here.
  - Fixed constants: add `fade_in_min_s: "0.1"`, `fade_in_max_s: "10"`,
    `fade_out_min_s: "0.1"`, `fade_out_max_s: "60"` and `fade_step_s: "0.1"`. Comment:
    "ranges and step of Fade-in time and Fade-out time (002/FR-001)".
  - Shared expressions, below `strip_level`, with one comment ("the current fade times in ms;
    the one conversion each, constitution IV.2"):
    - `fade_in_now_ms: "((uint32_t) lroundf(id(fade_in_time).state * 1000.0f))"`;
    - `fade_out_now_ms: "((uint32_t) lroundf(id(fade_out_time).state * 1000.0f))"`.
- [X] T014 [US2] In the range-check lambda of `esphome: on_boot:` in `esphome/buitenlamp.yaml`,
  replace the `fade_in_ms` and `fade_out_ms` checks (research S1, S9).
  - `static_assert(${fade_in_s} >= ${fade_in_min_s} && ${fade_in_s} <= ${fade_in_max_s},
    "fade_in_s must be ${fade_in_min_s}-${fade_in_max_s} s");`
  - `static_assert(${fade_out_s} >= ${fade_out_min_s} && ${fade_out_s} <= ${fade_out_max_s},
    "fade_out_s must be ${fade_out_min_s}-${fade_out_max_s} s");`
- [X] T015 [US2] In section 4 of `esphome/buitenlamp.yaml`, set the strip's
  `default_transition_length: ${fade_in_s}s` (research S7).
  - Comment: this is only the build-time value. The Fade-in time setting replaces it at
    runtime, also at boot (T016), so Home Assistant's own brightness and colour changes follow
    the chosen fade-in time (002/FR-008).
- [X] T016 [US2] Add Fade-in time and Fade-out time to the `number:` list in section 6 of
  `esphome/buitenlamp.yaml` (data-model.md, research S1, S7).
  - If US1 is not built yet, create the `number:` list first, as described in T009.
  - Both entities have the same shape as `on_period` (T009), with the same comments about
    refusal, storing within 0.5 s and the storage key:
    - `platform: template`, `entity_category: config`, `mode: box`,
      `unit_of_measurement: s`, `step: ${fade_step_s}`, `optimistic: true`,
      `restore_value: true`.
  - Fade-in time:
    - `id: fade_in_time`, `name: "Fade-in time"`;
    - `min_value: ${fade_in_min_s}`, `max_value: ${fade_in_max_s}`,
      `initial_value: ${fade_in_s}`;
    - `on_value:`
      - a `lambda:` with `id(strip).set_default_transition_length(${fade_in_now_ms});`;
      - then log `t=%lu fade-in time %.1f s` with `x`;
    - comment: `on_value` also fires for the value restored at boot, so the strip always uses
      the chosen value;
    - label `# 002/FR-001, 002/FR-002, 002/FR-004, 002/FR-005, 002/FR-008`.
  - Fade-out time:
    - `id: fade_out_time`, `name: "Fade-out time"`;
    - `min_value: ${fade_out_min_s}`, `max_value: ${fade_out_max_s}`,
      `initial_value: ${fade_out_s}`;
    - `on_value:` logs `t=%lu fade-out time %.1f s` with `x`;
    - label `# 002/FR-001, 002/FR-002, 002/FR-004, 002/FR-005, 002/FR-008`.
- [X] T017 [US2] Make the fade scripts in `esphome/buitenlamp.yaml` read the settings when they
  start (data-model.md, Changed scripts; research S7).
  - In `lamp_fade_on`, replace `${fade_in_ms}` with `${fade_in_now_ms}` in the `fade_length_ms`
    lambda.
  - In `lamp_fade_off`, replace `${fade_out_ms}` with `${fade_out_now_ms}` in the same way.
  - The partial-fade rule (`fminf(1.0f, …)`) stays as it is (002/FR-008).
  - Add `002/FR-003, 002/FR-008, 002/FR-009` to both scripts' labels.
  - Comment on each `globals.set`: the length is read once, when the fade starts, so a running
    fade keeps its length (002/FR-003).
  - Search `esphome/buitenlamp.yaml` for `fade_in_ms` and `fade_out_ms`. Nothing may remain.
- [X] T018 [US2] Run the compile gate on `esphome/buitenlamp.yaml` and upload over WiFi (see
  Conventions).
  - First do the range check from section 1 of `specs/002-ha-tunable-settings/quickstart.md`
    without editing the file.
  - `.venv\Scripts\esphome -s fade_out_s 61 compile esphome/buitenlamp.yaml` must stop with
    the `fade_out_s` `static_assert` message.
  - **Status (2026-11-09)**: config valid; compile passed, no warnings; RAM 34.3 % (110086
    bytes), flash 47.7 % (875740 bytes). The range check stopped with "static assertion
    failed: fade_out_s must be 0.1-60 s". **Upload not done** (see T011).

### Validation for User Story 2

- [ ] T019 [US2] Hands-on: run the fade parts of the scenarios in section 2 of
  `specs/002-ha-tunable-settings/quickstart.md`. Record the results in sections 4 and 5.
  - **W1, fade rows**: right after the first upload with US2, the card shows Fade-in time
    1.0 s and Fade-out time 5.0 s. Record the entity ids.
  - **W2, fades**: Fade-in time 20 and Fade-out time 0 are refused.
  - **W4**: the pairs 0.5/0.5, 3/10 and 10/60, 5 trials each (SC-003).
    - Then check Home Assistant on/off and a colour change with 3/10.
    - Watch the strip after dark for steps or flicker. Write the result for the 60 s fade-out
      into the "Smoothness at Fade-out time 60 s" row of section 5 (research S10).
    - Steps at the lowest levels at the end of fade-outs longer than 10 s are allowed (FR-009);
      record what is seen. Flicker, or visible steps anywhere else, fail the scenario.
  - **W5, second half**: change Fade-out time during a running 60 s fade-out.
  - **Power cut, once**: set Fade-in time to 2.0 s and Fade-out time to 8.0 s; cut the power
    2–5 s after the second change and restore it. The card shows both values and the next
    fades use them.
  - Test during the day as described in T012.

**Checkpoint**: the on period and both fade times can be set from Home Assistant.

---

## Phase 5: User Story 3 - Owner chooses when the lamp reacts to motion (Priority: P2)

**Goal**: Home Assistant shows a daily time window (start and end) and a Reaction mode:
`only when dark` (default, as before), `only in time window`, `in time window and dark`, or
`always`.
- Motion only counts while the mode's conditions are met.
- When they stop being met, the running on period finishes normally.
- In time window shows whether the window is active now.
- With the time unknown, motion always counts.

**Independent Test**: in daylight, choose Reaction mode `only in time window` with the window
ending 5 minutes from now.
- Wave before the end time: the lamp turns on.
- After the end time, let the light go out and wave again: the strip stays off, Motion is still
  reported, and the log says `motion ON, ignored (not allowed)`.
- In time window turned off at the end time.

### Implementation for User Story 3

- [X] T020 [US3] Add the reaction-mode and window settings to `substitutions:` of
  `esphome/buitenlamp.yaml` (contracts/settings.md, research S2, S3, S6).
  - Owner settings:
    - `window_start_default: "00:00:00"` and `window_end_default: "00:00:00"`. Comment:
      "default time window, used until it is chosen in Home Assistant; HH:MM:SS, seconds
      ignored; equal start and end means no restriction (002/FR-001, 002/FR-010)";
    - `reaction_mode_default: "only when dark"`. Comment: "default Reaction mode, used until it
      is chosen in Home Assistant; one of the four options of `reaction_mode` (002/FR-001,
      002/FR-011)";
    - `timezone: "Europe/Amsterdam"`, or the zone noted in T003 if Home Assistant uses another
      one. Comment: "time zone of the installation, for the time window; an IANA zone name.
      MUST be the same zone as Home Assistant's (Settings → System → General), which the lamp
      uses while connected (002/FR-010, research S6)".
  - Fixed constants: `rm_dark: "0"`, `rm_window: "1"`, `rm_window_and_dark: "2"` and
    `rm_always: "3"`.
    - Comment: "indexes of the Reaction mode options, in their fixed order. The lamp stores the
      index, so the order may never change (research S3)".
- [X] T021 [US3] In section 3 of `esphome/buitenlamp.yaml`, add `timezone: ${timezone}` to
  **both** time sources, `sntp_time` and `ha_time` (research S6).
  - Comments:
    - without it, ESPHome uses the time zone of the PC that builds the firmware;
    - each time source installs its own zone as the one global zone, so both must name the
      same one;
    - Home Assistant 2026.3 and newer send their own zone, which the lamp uses while connected,
      so `timezone` must match Home Assistant's;
    - ESPHome follows the zone's summer/winter time rule, so the window stays in local time
      (002/FR-010).
  - Label `# 002/FR-010` on both.
- [X] T022 [US3] Add the `datetime:` list to section 6 of `esphome/buitenlamp.yaml`, after
  `number:`, with Time window start and Time window end (data-model.md, research S2).
  - If neither US1 nor US2 is built yet, add the `# Settings chosen in Home Assistant` comment
    above this list.
  - Both entities: `platform: template`, `type: time`, `entity_category: config`,
    `optimistic: true`, `restore_value: true`.
  - `id: window_start`, `name: "Time window start"`, `initial_value: ${window_start_default}`.
  - `id: window_end`, `name: "Time window end"`, `initial_value: ${window_end_default}`.
  - `on_value:` on each logs `t=%lu window start %02u:%02u` (or `window end`) with
    `(unsigned) id(window_start).hour` and `(unsigned) id(window_start).minute` (or
    `window_end`).
  - Comments:
    - only hour and minute are used, so the seconds Home Assistant sends are ignored (minute
      precision, 002/FR-001);
    - the end minute is the first minute after the window;
    - the value is stored only when it changes, within 0.5 s (`preferences:`, 002/FR-004);
    - the names are the storage keys (contracts/ha-api.md).
  - Label `# 002/FR-001, 002/FR-004, 002/FR-005, 002/FR-010`.
- [X] T023 [US3] Add the `select:` list to section 6 of `esphome/buitenlamp.yaml`, after
  `datetime:`, with Reaction mode (data-model.md, research S3).
  - `platform: template`, `id: reaction_mode`, `name: "Reaction mode"`,
    `entity_category: config`, `optimistic: true`, `restore_value: true`.
  - `options:`, exactly in this order: `"only when dark"`, `"only in time window"`,
    `"in time window and dark"`, `"always"`.
  - `initial_option: ${reaction_mode_default}`.
  - `on_value:` logs `t=%lu reaction mode %s` with
    `id(reaction_mode).current_option().c_str()`.
    - This runs only on a change, so reading the text is allowed here.
    - If `current_option()` gives no `c_str()` in 2026.10.1, log the index instead:
      `(unsigned) id(reaction_mode).active_index().value_or(${rm_dark})` with `%u`.
  - Comments:
    - the lamp stores the index of the chosen option. Never reorder or insert options; only
      append, and update the `rm_*` constants to match;
    - the value is stored only when it changes, within 0.5 s (`preferences:`, 002/FR-004).
  - Label `# 002/FR-001, 002/FR-004, 002/FR-005, 002/FR-011`.
- [X] T024 [US3] Add `in_window` to the `binary_sensor:` list in section 6 of
  `esphome/buitenlamp.yaml`, under `# Home Assistant entities`, after `daylight`
  (data-model.md, Window rule; research S2, S5).
  - `platform: template`, `id: in_window`, `name: "In time window"`. No device class.
  - The lambda, one calculation, with t, s and e in minutes since midnight:

    ```cpp
    const auto now = id(sntp_time).now();
    if (!now.is_valid()) {
      return {};
    }
    const int t = now.hour * 60 + now.minute;
    const int s = id(window_start).hour * 60 + id(window_start).minute;
    const int e = id(window_end).hour * 60 + id(window_end).minute;
    if (s == e) {
      return true;
    }
    if (s < e) {
      return t >= s && t < e;
    }
    return t >= s || t < e;
    ```

  - Comments:
    - no state while the time is unknown (002/FR-014);
    - `60` is minutes per hour;
    - equal start and end means no restriction; with an end before the start, the window
      crosses midnight (002/FR-010);
    - the lambda runs on every loop pass and only changes are sent, so the status follows a
      minute boundary or a settings change within one pass (002/FR-017).
  - Label `# 002/FR-010, 002/FR-017`.
- [X] T025 [US3] Add `motion_allowed` to the `binary_sensor:` list in section 6 of
  `esphome/buitenlamp.yaml`, under `# Internal inputs`, after `pir_raw` (data-model.md, When
  motion is allowed; research S4).
  - `platform: template`, `id: motion_allowed`, `name: "Motion allowed"`, `internal: true`.
  - The lambda, one decision:

    ```cpp
    if (!id(daylight).has_state() || !id(in_window).has_state()) {
      return true;
    }
    const bool dark = !id(daylight).state;
    const bool inside = id(in_window).state;
    switch (id(reaction_mode).active_index().value_or(${rm_dark})) {
      case ${rm_window}:
        return inside;
      case ${rm_window_and_dark}:
        return inside && dark;
      case ${rm_always}:
        return true;
      case ${rm_dark}:
      default:
        return dark;
    }
    ```

  - It reads `active_index()`, not the option text, so no string is compared on every loop
    pass (constitution III.1).
  - `on_release:` runs `- script.execute: conditions_ended`.
  - Comments:
    - the one place where the reaction mode is decided (constitution IV.2);
    - with the time unknown, motion is allowed in every mode (002/FR-014);
    - it starts ON at boot, and ESPHome fires no trigger for the first state;
    - `on_release` fires for every way the conditions end: the window ends, sunrise in a mode
      that uses darkness, a changed mode or window, or the time becoming known while motion is
      not allowed (data-model T13).
  - Label `# 002/FR-011, 002/FR-012, 002/FR-014`.
- [X] T026 [US3] Rename the script `dark_period_ended` to `conditions_ended` in
  `esphome/buitenlamp.yaml`, and remove `daylight`'s trigger (data-model T13; research S4).
  - Keep the body. Change its log line to `t=%lu conditions ended, motion no longer counted`;
    W10 and W17 look for this text.
  - Label becomes `# FR-018, FR-023, 002/FR-012`.
  - Replace the comment:
    - a counted motion still going when the conditions of the Reaction mode end, ends there;
    - so the on period runs from that moment, and the light fades out normally even if the
      person keeps moving (002/FR-012);
    - triggered by `motion_allowed` turning off.
  - Add one more comment line. Unlike 1.1.0, this also runs when the time first becomes known
    while motion is not allowed. A counted motion from the unknown-time period then ends one
    on period later (research S4, accepted behaviour change).
  - In `daylight`, remove `on_press:`. Replace its comment: no state means the time is
    unknown; `motion_allowed` then allows motion (002/FR-014). Add `002/FR-011` to its label.
  - Search the file for `dark_period_ended`. Nothing may remain.
- [X] T027 [US3] In the script `motion_started` in `esphome/buitenlamp.yaml`, use
  `motion_allowed` instead of Daylight (data-model T1, T3, T5; research S4).
  - In the `and:` condition, replace `- not: binary_sensor.is_on: daylight` with
    `- binary_sensor.is_on: motion_allowed`. Keep the `need_fresh_motion` check.
  - In the `else:` log, replace `"daylight"` with `"not allowed"`, so ignored motion logs
    `motion ON, ignored (not allowed)`. W9 looks for this text.
  - Update the comments:
    - "Daylight not ON …" becomes "motion_allowed: the conditions of the Reaction mode are
      met, or the time is unknown";
    - the header: a motion that starts while motion is not allowed never counts, even if it
      lasts until motion is allowed (002/FR-013).
  - Add `002/FR-011, 002/FR-013, 002/FR-016` to the label. FR-016: the `motion` entity reports
    the motion whatever this script decides.
- [X] T028 [US3] In the script `fallback_timer` in `esphome/buitenlamp.yaml`, use
  `motion_allowed` instead of Daylight (data-model T10; research S4).
  - In the inner `and:` condition, replace `- not: binary_sensor.is_on: daylight` with
    `- binary_sensor.is_on: motion_allowed`.
  - Change the log to `t=%lu stays on (recent motion, motion allowed)`.
  - Change the comment "Recent motion in the dark keeps the light on" to "Recent motion keeps
    the light on only while motion is allowed".
  - Add `002/FR-015` to the label.
  - Search the file for `daylight`. It may only appear in its own definition and in the
    `motion_allowed` lambda.
- [X] T029 [US3] Run the compile gate on `esphome/buitenlamp.yaml` and upload over WiFi (see
  Conventions).
  - **Status (2026-11-09)**: config valid; compile of the full 1.2.0 at DEBUG passed, no
    warnings from `esphome/buitenlamp.yaml` (only ESP-IDF's own CMake notices); RAM 34.4 %
    (110566 bytes), flash 48.1 % (882638 bytes). **Upload not done**: it needs the real API
    key in `esphome/secrets.yaml` (see T002). Upload this build before the hands-on tasks:
    `.venv\Scripts\esphome run esphome/buitenlamp.yaml --device kantoorlamp.local --no-states`.

### Validation for User Story 3

- [ ] T030 [US3] Hands-on: run the reaction-mode scenarios in section 3 of
  `specs/002-ha-tunable-settings/quickstart.md`, and record the results in sections 4 and 5.
  - **W1, remaining rows**: right after the first upload with US3, the card shows Time window
    start 00:00, Time window end 00:00 and Reaction mode `only when dark`, and the status
    shows In time window = on. Record the entity ids.
  - **W2, Reaction mode**: `select.select_option` with `never` is refused by Home Assistant;
    Reaction mode is unchanged.
  - **W9, W14 and W15**: 5 trials per mode, inside and outside the conditions (SC-005).
  - **W13**: start equals end.
  - **W19**: manual mode is not limited.
- [ ] T031 [US3] Hands-on: run the condition-change scenarios in section 3 of
  `specs/002-ha-tunable-settings/quickstart.md`, and record the results.
  - **W10**: the window ends during motion.
  - **W11**: the window starts during motion.
  - **W17**: the mode changes while the light is on.
  - **W21**: In time window follows within 2 s, 10 checks (FR-017). Also the other 6 SC-008
    checks: the card confirms a new window time (3 times) and a new Reaction mode (3 times)
    within 2 s.
  - **W18**: time unknown, including the counted motion that ends when the time becomes known.
  - **W20 (a) and (b)**: the fallback with the reaction mode. It uses the On period setting
    from US1.
  - Before W18, note that the lamp's log shows the restored settings at boot (the `on_value`
    lines).
- [ ] T032 [P] [US3] Hands-on: run W12 and W16 (a) of
  `specs/002-ha-tunable-settings/quickstart.md` over 3 different days (SC-006).
  - Use Reaction mode `only in time window`. On one of the days, use a window that crosses
    midnight.
  - Each day, check in the In time window history that it changed within 1 minute of the set
    local times.
  - Start it right after T031. The documentation tasks T034–T038 can run meanwhile. T039 and
    T040 change the window and Reaction mode, so they wait until it is finished.
- [ ] T033 [US3] Hands-on, deferred to 28 March 2027 (plan.md, Complexity Tracking): run W16 (b)
  of `specs/002-ha-tunable-settings/quickstart.md`.
  - Keep a window of 08:00–17:00 over the summer-time change.
  - Check that In time window changes at 08:00 and 17:00 local time on both days.
  - Write the date and result in the "W16 (b)" row of section 5.
  - Delivery does not wait for this task.

**Checkpoint**: all three user stories pass. With the default settings the lamp behaves as 1.1.0.

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: documentation, the review, traceability and resource gates, the full power-cut and
regression runs, and delivery.

- [X] T034 [P] Update `README.md` for 1.2.0.
  - Add a section **Settings in Home Assistant** after "Choosing brightness and colour". It says
    where the settings are: device page, section **Configuration**. It gives:
    - the six settings, with range, unit and default, from
      `specs/002-ha-tunable-settings/contracts/ha-api.md`;
    - the four reaction modes in one line each, spelled exactly as in Home Assistant;
    - the window rules: the end minute is outside; equal start and end means no restriction;
      an end before the start crosses midnight;
    - the In time window status;
    - that a change applies from the next on period or fade;
    - that a change is stored on the lamp within 2 seconds, so it is kept after a power cut,
      restart or update;
    - that the YAML values are only defaults.
  - In **Choosing brightness and colour**, extend "The lamp remembers them after a power cut or
    update" with: a choice is stored within 2 seconds.
  - In **Step 2**, add `timezone`: it must be the same zone as Home Assistant's (Settings →
    System → General).
  - Update **Changing a setting later**:
    - the six settings above are changed in Home Assistant;
    - only the other settings need an upload;
    - converting a 1.1.0 configuration: divide `on_period_ms`, `fade_in_ms` and `fade_out_ms`
      by 1000 and enter them as `on_period_s`, `fade_in_s` and `fade_out_s`.
  - Update **Step 5**:
    - in the story 5 row, replace "Set `on_period_ms` back to `"60000"`" with "Set
      `on_period_s` back to `"60"`";
    - add a row for feature 002: W1–W22 from `specs/002-ha-tunable-settings/quickstart.md`;
    - add a tip: with Reaction mode `always`, timing tests can run in daylight.
  - In **Step 6**, the project version is `1.2.0`.
  - Replace every `--device buitenlamp.local` with `--device kantoorlamp.local`.
- [X] T035 [P] Point the 001 documents at the changes of 1.2.0.
  - At the top of `specs/001-pir-ledstrip-fade/contracts/settings.md`, add a note: from 1.2.0,
    `fade_in_ms`, `on_period_ms` and `fade_out_ms` are `fade_in_s`, `on_period_s` and
    `fade_out_s`, and are only defaults of Home Assistant settings. Link
    `specs/002-ha-tunable-settings/contracts/settings.md`.
  - In `specs/001-pir-ledstrip-fade/research.md`, after the "Remembering it (FR-031)" bullet
    of R18, add a note: "*Superseded in part on 2026-11-09 (002 research S8)*: from 1.2.0,
    `flash_write_interval` is 500 ms, so a chosen brightness and colour is in flash within
    1.5 s instead of within 60 s." Link `specs/002-ha-tunable-settings/research.md`.
  - In `specs/001-pir-ledstrip-fade/quickstart.md`:
    - section 5: replace `--device buitenlamp.local` with `--device kantoorlamp.local` in the
      upload and log commands;
    - V21: add a note with the same meaning as the settings note: from 1.2.0, set
      `on_period_s` to `"120"`. The YAML value only applies while no On period has been chosen
      in Home Assistant.
  - In `specs/001-pir-ledstrip-fade/contracts/ha-api.md`, update the Device table and the text
    below it to the rename of commit e1dc5dc:
    - host name: default `kantoorlamp` → `kantoorlamp.local`;
    - friendly name: default `Buitenlamp (kantoor)`;
    - "Discovered: Buitenlamp (kantoor)".
- [X] T036 Review gate: check `esphome/buitenlamp.yaml` against constitution v2.0.0, focusing on
  what this feature changed (plan.md, Constitution Check).
  - Every changed or new script, automation and entity has the labels of data-model.md, with
    the `002/` prefix for this feature (IV.3).
  - The reaction rule exists only in `motion_allowed`, and the window rule only in `in_window`.
    "Motion counts" is decided only in `motion_started` (IV.2).
  - The seconds-to-ms conversions exist only in the three `*_now_ms` substitutions (IV.2).
  - No lambda that runs every loop pass compares or builds a string (III.1):
    `current_option()` appears only in the select's `on_value`.
  - Flash writes (III.3):
    - `restore_value: yes` appears only on the six `chosen_*` globals of 001, and
      `restore_value: true` only on the six setting entities of this feature;
    - `preferences:` appears once, with `flash_write_interval: ${flash_write_interval}`;
    - no lambda calls `global_preferences->sync()`.
  - `1000.0f` appears only in the `*_now_ms` substitutions, `60` only in `in_window`, and
    `500ms` only in the `flash_write_interval` substitution. No other new numeric literals
    (V.1).
  - `millis()` appears only in log lines (II.2).
  - No `web_server`, `captive_portal`, `ap:`, `effects:` or external components (V.4, V.5).
  - `strip` still has `restore_mode: ALWAYS_OFF` (I.1).
  - Record any new deviation in the Complexity Tracking table of
    `specs/002-ha-tunable-settings/plan.md`.
  - **Status (2026-11-09)**: every check passed; no new deviation, so Complexity Tracking is
    unchanged.
- [X] T037 [P] Traceability gate: in section 6 of `specs/002-ha-tunable-settings/quickstart.md`,
  check each row of the "Implemented by" column against the ids actually used in
  `esphome/buitenlamp.yaml`, and correct any that differ.
  - FR-004 must name `restore_value: true` on the six entities and the `preferences:` block.
  - FR-010 must name both `sntp_time` and `ha_time`.
  - FR-012 must name `conditions_ended`.
  - **Status (2026-11-09)**: all 17 rows match the ids in the YAML; nothing to correct.
- [X] T038 Bring the design docs in line with what was built. Do this after T036, which may
  also edit plan.md.
  - Record in `specs/002-ha-tunable-settings/research.md`, `plan.md`, `data-model.md` or
    `contracts/` any name or behaviour of the build that differs from them, for example a key
    renamed under the Conventions rule.
  - In `specs/002-ha-tunable-settings/data-model.md`, list the log lines added on setting
    changes (the `on_value` logs of T009, T016, T022 and T023).
  - Research S6, S8 and the plan's source tree already describe the time zone on both sources
    and the `preferences:` block; change them only if the build differs.
  - **Status (2026-11-09)**: the build matches the design; no key was renamed. data-model.md
    has a new "Log lines" table with the setting-change lines and the three changed lines.
- [ ] T039 Hands-on: run W6 in full (3 power-cut trials, SC-004), W7 and W8 from section 2 of
  `specs/002-ha-tunable-settings/quickstart.md`. Keep the DEBUG log open.
  - W6 steps (a)–(e), three times. Each trial uses values that differ from the trial before,
    because an unchanged value is not written and so proves nothing:

    | Trial | On period | Fade-in | Fade-out | Window | Reaction mode | Light | Wave when |
    |-------|-----------|---------|----------|--------|---------------|-------|-----------|
    | 1 | 90 s | 2.0 s | 8.0 s | 07:30–22:15 | `in time window and dark` | 30 % red | after sunset, before 22:15 |
    | 2 | 95 s | 2.5 s | 9.0 s | 07:45–22:30 | `only in time window` | 40 % blue | between 07:45 and 22:30 |
    | 3 | 100 s | 3.0 s | 10.0 s | 08:00–22:45 | `always` | 50 % green | any time |

    - set the six settings to the trial's values, choose the trial's brightness and colour on
      the Buitenlamp light, then switch it off;
    - after each change, check the `[preferences] Writing … written` line: within 0.5 s for a
      setting, within 1.5 s for brightness and colour;
    - cut the power 2–5 s after the last change, and restore it;
    - check that the card shows the trial's values, and that one wave keeps the light on for
      the trial's on period (+1 s) after Motion turns off, in the trial's colour;
    - wait until at least 1 minute after the power returned (ESPHome then clears its safe-mode
      boot counter, which logs one `Writing` line), then leave the lamp alone for 2 min: no
      `Writing` line may appear.
  - W6 step (f), once: upload with a different default instead of editing the file:
    `.venv\Scripts\esphome -s on_period_s 30 run esphome/buitenlamp.yaml --device
    kantoorlamp.local --no-states`. The chosen On period (100 s after trial 3) must stay.
    T041's upload puts the file's default back.
  - Record the results in sections 4 and 5.
- [ ] T040 Hands-on: run W22, the regression check with default settings (SC-007).
  - Set all six settings back to the W1 values.
  - Run V3, V4, V8, V9, V10 and V11 from `specs/001-pir-ledstrip-fade/quickstart.md`.
  - Run them as often as 001's Trials table asks (001 quickstart section 8), because SC-007
    depends on 001's success criteria:
    - V3 and V4 (one wave gives both): 10 trials (001/SC-001, 001/SC-002);
    - V8: 10 trials (001/SC-009);
    - V10: 3 days (001/SC-010);
    - V11: 5 trials (001/SC-011);
    - V9: once.
  - V10 only reads the Daylight history, which no setting changes, so its 3 days may overlap
    with T032.
  - Record the results in section 5 of `specs/002-ha-tunable-settings/quickstart.md`.
- [ ] T041 Prepare the delivered firmware. Do this after T037, which edits the same quickstart
  file, and after T039 and T040, which need the DEBUG log.
  - Run the compile gate at DEBUG and record RAM and flash use (% and bytes).
  - Set `log_level` back to `"INFO"` in `esphome/buitenlamp.yaml`, run the compile gate again
    and record RAM and flash.
  - Both go in the "Flash usage" and "Static RAM usage" rows of section 5 of
    `specs/002-ha-tunable-settings/quickstart.md`. Limits: RAM ≤ 75 %, flash ≤ 90 %.
  - Upload over WiFi.
  - **Status (2026-11-09)**: done early, at the owner's request, before T030–T040. `log_level`
    is `"INFO"`; config valid; compile passed, no warnings from `buitenlamp.yaml`. INFO: RAM
    34.4 % (110566 bytes), flash 47.5 % (872278 bytes). DEBUG (T029 build): RAM 34.4 %
    (110566 bytes), flash 48.1 % (882638 bytes). Recorded in section 5. Still open: the
    upload. For the tests that read the DEBUG log (T039, T040 and the timing scenarios), set
    `log_level` to `"DEBUG"` temporarily and back to `"INFO"` afterwards.
- [ ] T042 Hands-on: smoke-test the INFO firmware and finish section 5 of
  `specs/002-ha-tunable-settings/quickstart.md`.
  - After dark, with the default settings: wave once, time the fade-in (about 1 s), the on
    period (60 s after Motion turns off in Home Assistant, allowing up to 2 s) and the fade-out
    (about 5 s). Switch on and off once from Home Assistant.
  - Change On period once from a phone, check that it is used, and set it back.
  - Fill in the remaining rows:
    - "Scenarios W1–W22 passed", listing any exceptions (W16 (b) is deferred to T033);
    - the date and `project: version` `1.2.0`.
  - Set back any test settings the owner does not want to keep.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: no dependencies. Do T001 first, so every edit lands on the feature
  branch. T002 and T003 can then run together.
- **Foundational (Phase 2)**: needs T002, a passing baseline build. T004–T006 edit the same
  file and run in order.
- **US1 (Phase 3)**: needs Foundational. Its validation (T012) also needs T003. Its power-cut
  check needs T006.
- **US2 (Phase 4)**: needs Foundational. It does not need US1. If US1 is not built, T016 creates
  the `number:` list.
- **US3 (Phase 5)**: needs Foundational. It does not need US1 or US2, with one exception: W20
  (T031) sets the On period in Home Assistant, which needs US1.
- **Polish (Phase 6)**: needs all three stories.
  - T038 after T036, because both may edit plan.md.
  - T041 after T037, because both edit quickstart.md, and after T039 and T040, which need the
    DEBUG log.
  - T042 after T041.
  - T039 and T040 need all three stories uploaded.
  - T039 and T040 after T032, because they change the window and Reaction mode that T032
    watches (only V10 of T040 may overlap with T032).
- **T033** (W16 (b)) runs on 28 March 2027 and does not block delivery.

### User Story Dependencies

- **US1 (P1)**: independent. MVP.
- **US2 (P2)**: independent of US1 and US3 in behaviour. It shares the `number:` list with US1.
- **US3 (P2)**: independent of US1 and US2 in behaviour. It changes `motion_started`,
  `fallback_timer`, `daylight` and the renamed `conditions_ended`, which US1 and US2 do not
  touch.

### Within Each User Story

1. Edit `esphome/buitenlamp.yaml` in task order. Each task builds on the one before.
2. Run the compile gate, the range check where the story has one, and upload over WiFi.
3. Run the story's hands-on scenarios.

### Parallel Opportunities

- Setup: T002 (build) and T003 (hands-on) do not touch the same files.
- T032 (W12 and W16 (a), three days) runs alongside T034–T038, which do not touch the lamp.
- Polish: T034 (`README.md`), T035 (001 documents), T036 (review, reads the YAML) and T037
  (002 quickstart section 6) touch different files.
- Every other task edits `esphome/buitenlamp.yaml` and must run in order. Two agents editing
  that file at the same time would conflict.
- The stories are independent in behaviour but not in files, so build them one after another.

---

## Parallel Example: Setup and Polish

```text
# Setup, after T001:
Task: "T002 Baseline compile gate on the unchanged esphome/buitenlamp.yaml"
Hands-on: "T003 Lamp runs 1.1.0 and is known to Home Assistant"

# After T031, alongside T034–T038:
Hands-on: "T032 W12 and W16 (a): In time window history on 3 days"

# Polish, all at once:
Task: "T034 Update README.md for 1.2.0"
Task: "T035 Point the 001 settings contract, research R18 and quickstart at 1.2.0"
Task: "T036 Review gate on esphome/buitenlamp.yaml"
Task: "T037 Check the traceability table in 002 quickstart.md section 6"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup.
2. Complete Phase 2: Foundational (CRITICAL: blocks all stories).
3. Complete Phase 3: User Story 1.
4. **STOP and VALIDATE**: T012. The owner can set the on period from a phone, and it survives a
   power cut seconds later. The fade times, window and reaction mode are still the 1.1.0
   behaviour.

### Incremental Delivery

1. Setup and Foundational: version 1.2.0 builds, at DEBUG, on the feature branch, and stores
   changes within 2 s (this already applies to 001's brightness and colour).
2. US1: On period from Home Assistant (MVP).
3. US2: Fade-in time and Fade-out time from Home Assistant.
4. US3: time window, Reaction mode and In time window. With the defaults, behaviour is
   unchanged.
5. Polish: README, 001 pointers, the gates, the full power-cut and regression runs, INFO
   firmware.

Every phase ends with a compile, a WiFi upload and its scenarios, so the lamp always runs a
working firmware. With the default settings, each intermediate version behaves like 1.1.0.

---

## Notes

- Keep `log_level` at `DEBUG` until T041. The timing scenarios read the `t=` timestamps from the
  log, and the power-cut trials read the `[preferences] Writing` lines.
- Always watch the log with `--no-states`. Otherwise the log viewer counts as a Home Assistant
  client and holds off the manual-mode fallback (W20).
- For out-of-range tests, use Developer tools → Actions (`number.set_value`). The card itself
  does not accept such values.
- Mark each task `[X]` when it is done. Commit after each phase, if the owner asks for commits.
