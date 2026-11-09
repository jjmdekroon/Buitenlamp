# Buitenlamp

An outdoor LED strip that fades on when someone walks past after dark. An ESP32-C3 running
[ESPHome](https://esphome.io) reads a PIR motion sensor and drives an SK6812 RGBW strip.

- After dark, motion fades the strip on in 1 s. It stays on until 60 s after the last motion,
  then fades off over 5 s. These are the defaults: the on period, both fade times, a daily
  time window and when the lamp reacts can be changed in Home Assistant.
- Darkness is worked out from sunset and sunrise at your location. There is no light sensor.
- Home Assistant shows the lamp as Buitenlamp (a light with on/off, brightness and colour),
  Motion, Daylight, In time window, Brightness and Mode, plus six settings. The brightness and
  colour you choose are used every time the lamp turns on, by motion too, and are remembered
  after a power cut.
- The lamp keeps working without WiFi or Home Assistant.
- The six settings above are changed in Home Assistant. Every other setting is in one block at
  the top of `esphome/buitenlamp.yaml`: to change it, edit the value and upload over WiFi.

## What's in this repository

| Path | What it is |
|------|------------|
| [esphome/buitenlamp.yaml](esphome/buitenlamp.yaml) | the whole device: settings at the top, then the logic |
| [esphome/secrets.example.yaml](esphome/secrets.example.yaml) | template for your WiFi details and encryption key (committed) |
| `esphome/secrets.yaml` | your own copy with the real values (never committed) |
| [requirements.txt](requirements.txt) | the pinned ESPHome version, 2026.10.1 |
| [specs/001-pir-ledstrip-fade/](specs/001-pir-ledstrip-fade/) | spec, plan, wiring and the full test procedure |
| [specs/002-ha-tunable-settings/](specs/002-ha-tunable-settings/) | the settings in Home Assistant (version 1.2.0): spec, plan and tests |

## Before you start

You need:
- a Windows PC with Python 3.13;
- the parts in the bill of materials in
  [hardware.md](specs/001-pir-ledstrip-fade/contracts/hardware.md#bill-of-materials);
- Home Assistant on the same network, if you want to use it.

Run every command below in PowerShell from the repository folder.

Install ESPHome once (skip this if the `.venv` folder already exists):

```powershell
python -m venv .venv
.venv\Scripts\python -m pip install -r requirements.txt
.venv\Scripts\esphome version
```

The last command should print `Version: 2026.10.1`.

## Setting up the lamp

Do the six steps in order. The task numbers in brackets refer to
[tasks.md](specs/001-pir-ledstrip-fade/tasks.md), so you can tick them off there.

### Step 1: Add your WiFi details (T005)

1. Open `esphome/secrets.yaml`. If it does not exist (for example, in a fresh clone):
   1. Copy `esphome/secrets.example.yaml` to `esphome/secrets.yaml`.
   2. Generate an encryption key and paste it as `api_encryption_key`:

      ```powershell
      python -c "import secrets,base64;print(base64.b64encode(secrets.token_bytes(32)).decode())"
      ```

2. Fill in `wifi_ssid` and `wifi_password`. Use a 2.4 GHz network, because the ESP32-C3 cannot
   join a 5 GHz one.
3. Keep `api_encryption_key` at hand. You paste it into Home Assistant in step 4, and it is
   also the password for updates over WiFi.

Put real values only in `secrets.yaml`, which git ignores. `secrets.example.yaml` is committed,
so it should only ever hold placeholders.

### Step 2: Set your location and strip length (T010)

In the `substitutions:` block at the top of `esphome/buitenlamp.yaml`, set these values:

| Setting | What to enter |
|---------|---------------|
| `latitude`, `longitude` | Where the lamp is installed, in decimal degrees, north and east positive. The values in the file are an example (Utrecht). Darkness is worked out from these, so wrong values make the lamp react at the wrong times. In OpenStreetMap or Google Maps, right-click the spot to copy its coordinates. |
| `num_leds` | The number of pixels on your strip, 1–240. |
| `timezone` | Your time zone, for the time window, for example `"Europe/Amsterdam"`. It must be the same zone as Home Assistant's (Settings → System → General), because the lamp uses Home Assistant's zone while it is connected. |
| `log_level` | `"DEBUG"` for now. The tests in step 5 read timestamps from the debug log. Step 6 sets it back to `"INFO"`. |

Then check that the firmware builds:

```powershell
.venv\Scripts\esphome config esphome/buitenlamp.yaml
.venv\Scripts\esphome compile esphome/buitenlamp.yaml
```

- `config` must end with `Configuration is valid!`.
- The first `compile` downloads ESP-IDF, which takes several minutes.
- If a setting is out of range, the compile stops with a message that names the setting and
  its allowed range.

### Step 3: Build the hardware (T006)

Wire everything as shown in
[hardware.md](specs/001-pir-ledstrip-fade/contracts/hardware.md). In short:

| From | To | Why |
|------|----|-----|
| PIR **VCC** | board **3V3** | **Never 5 V**: the sensor's signal would then exceed 3.3 V |
| PIR **SIG** | **GPIO4** (XIAO D2) | motion input |
| **GPIO5** (XIAO D3) | 74AHCT125 **1A** (pin 2), plus a **10 kΩ** resistor to GND | keeps the data line low while the board boots |
| 74AHCT125 **1Y** (pin 3) | **330 Ω** resistor → strip **DIN** | lifts the data signal to the 5 V the strip needs |
| 74AHCT125 **VCC** (14), **1OE** (1), **GND** (7) | 5 V, GND, GND | powers and enables the buffer |
| 74AHCT125 unused **2A, 3A, 4A** (5, 9, 12) | GND | unused inputs must not float |
| 74AHCT125 unused **2OE, 3OE, 4OE** (4, 10, 13) | 5 V | disables the unused outputs |
| Strip **5V** and **GND** | fused 5 V supply, with a **1000 µF** capacitor across them at the feed point | absorbs the inrush current at power-on |
| Every GND | one common ground point | |

**Power supply**: at least `num_leds × 0.080 A × 1.2 + 0.3 A`. Any colour can be chosen in
Home Assistant, so all four channels (R, G, B and W) can be at full at the same time. For
example, 26 LEDs need 2.8 A, so use a supply of 3 A or more. Use a fuse rated at about 1.25 ×
the calculated current.

Before you apply power, check that:
- PIR VCC goes to **3V3**, not 5 V;
- the 74AHCT125 is powered from **5 V**.

While the board is plugged into the PC over USB, disconnect the external 5 V from the board's
5V pin, unless your board has a protection diode there. Otherwise the two supplies can fight
each other.

### Step 4: First upload and add the lamp to Home Assistant (T012, T013)

1. Connect the board to the PC with USB. Find its port in Device Manager, under
   Ports (COM & LPT).
2. Build, upload over USB and show the log, in one command. Replace `COM5` with your port:

   ```powershell
   .venv\Scripts\esphome run esphome/buitenlamp.yaml --device COM5
   ```

   Check in the log that the board starts and joins WiFi. Press Ctrl+C to stop the log.
3. From now on, build and upload over WiFi:

   ```powershell
   .venv\Scripts\esphome run esphome/buitenlamp.yaml --device kantoorlamp.local --no-states
   ```

   To only watch the log, without uploading:

   ```powershell
   .venv\Scripts\esphome logs esphome/buitenlamp.yaml --device kantoorlamp.local --no-states
   ```

   Always add `--no-states` when you watch the log over WiFi. Without it, the log viewer counts
   as Home Assistant, which delays the fallback from manual mode.

> **Use `run`, not `upload`, after every change to the YAML.** `esphome run` builds the
> firmware, uploads it and shows the log. `esphome upload` only sends the firmware that was
> built last, so your change would not reach the board. The first log lines show when the
> firmware on the board was built ("compiled on …").
4. In Home Assistant, open Settings → Devices & services. You should see **New device
   discovered: Buitenlamp (kantoor)**. Click **Add** and paste `api_encryption_key`. You do
   not need to type an address. Note how long discovery took, because test V13 needs it.

The device page now shows Buitenlamp (light), Motion, Daylight, In time window, Brightness and
Mode, and the six settings under **Configuration** (see below).

**Updating from version 1.0.0?** Version 1.0.0 had a switch called Buitenlamp; 1.1.0 replaces
it with a light of the same name. After the update, the old switch shows as unavailable.
Delete it on the device page, and change any automations or dashboards from
`switch.buitenlamp` to `light.buitenlamp`.

### Choosing brightness and colour

1. In Home Assistant, open the Buitenlamp light and choose a brightness and a colour. The
   white slider controls the strip's separate white LEDs.
2. The lamp fades to that brightness and colour and goes to **manual** mode, just like
   switching it on. Every change you make in Home Assistant counts as "on".
3. Switch the lamp off in Home Assistant. It fades out and returns to **automatic** mode.

From then on, every "on" uses the chosen brightness and colour, both from motion and from Home
Assistant. The lamp remembers them after a power cut or update; a choice is stored within
2 seconds. The `on_*` values in `esphome/buitenlamp.yaml` are only the starting values until
you choose your own.

### Settings in Home Assistant

Open the lamp's device page in Home Assistant (Settings → Devices & services → ESPHome → the
lamp). The section **Configuration** holds six settings:

| Setting | Range | Default | What it does |
|---------|-------|---------|--------------|
| On period | 1–86400 s, steps of 1 s | 60 s | how long the light stays on after the sensor stops reporting motion |
| Fade-in time | 0.1–10 s, steps of 0.1 s | 1.0 s | time of a full fade from off to on; also used for brightness and colour changes from Home Assistant |
| Fade-out time | 0.1–60 s, steps of 0.1 s | 5.0 s | time of a full fade from on to off |
| Time window start | a time of day | 00:00 | first minute of the daily time window |
| Time window end | a time of day | 00:00 | first minute **after** the time window |
| Reaction mode | four choices, below | `only when dark` | when motion switches the lamp on |

The four reaction modes:
- `only when dark`: after sunset and before sunrise, whatever the time window. This is how
  the lamp worked before 1.2.0.
- `only in time window`: inside the time window, whether it is dark or light.
- `in time window and dark`: inside the time window and dark.
- `always`: at any time.

The time window:
- The end minute is outside the window: 08:00–17:00 covers 08:00 up to and including 16:59.
- Equal start and end, such as the default 00:00–00:00, means no restriction.
- An end before the start crosses midnight, for example 22:00–06:00.
- **In time window** (under the status entities) shows whether the window is active now.

Good to know:
- A change applies from the next on period or fade. A running one finishes with the value it
  started with.
- A change is stored on the lamp within 2 seconds, so it is kept after a power cut, restart or
  update.
- The matching values in `esphome/buitenlamp.yaml` (`on_period_s`, `fade_in_s`, `fade_out_s`,
  `window_start_default`, `window_end_default`, `reaction_mode_default`) are only the
  defaults. They are used until you choose a value in Home Assistant, and ignored after that.
- Switching the lamp on or off from Home Assistant always works, whatever the reaction mode.
- Until the lamp knows the time (for example after a power cut without WiFi), motion switches
  it on in every mode.

### Step 5: Test on the device (T024–T055)

The full procedure is in sections 7–9 of
[quickstart.md](specs/001-pir-ledstrip-fade/quickstart.md).

1. **Measure the sensor** (section 7). Power-cycle the board and watch the `PIR raw` lines for
   60 s without moving: the last false detection gives the warm-up time. Then wave once and
   keep still: the time from `motion ON` to `motion OFF` is the hold time. Write both into
   section 9.
2. **Run the scenarios** in section 8, story by story:

   | Story | Scenarios | Notes |
   |-------|-----------|-------|
   | 1. Fades on and off | V1, V2, V3, V4, V7, V12, V25 | Judge smoothness (V3, V7) after dark |
   | 2. Stays on while someone is there | V5 (30 min), V6, then V3 and V4 again | |
   | 3. Reacts only after dark | V8, V9, V11 with V19, V23, V27; V10 over 3 days | After V9 and V27, set `longitude` back and use `run` again |
   | 4. Home Assistant | V13, V14, V15, V16, V20; V17 (1 h); V18 | Watch the log with `--no-states` |
   | 5. Changing settings | V21, V24 | Set `on_period_s` back to `"60"` afterwards |
   | Feature 002: settings in Home Assistant | W1–W22 in [quickstart.md](specs/002-ha-tunable-settings/quickstart.md) | Power-cut trials: cut the power 2–5 s after the last change |

- The lamp ignores motion in daylight, so run the timing tests after dark. To test during the
  day, set `longitude` about 180° away from the real value, use `run`, and set it back
  afterwards (scenario V9).
- Tip: with Reaction mode `always` in Home Assistant, the timing tests can run in daylight too.
  Set it back to `only when dark` afterwards.
- Times come from the `t=` timestamps in the DEBUG log.
- Repeat each scenario as often as the Trials table in section 8 says. Record each result in
  section 9.

### Step 6: Deliver the final firmware (T059–T061)

1. In `esphome/buitenlamp.yaml`, set `log_level` back to `"INFO"`.
2. Compile, and check that no warnings point to `buitenlamp.yaml`:

   ```powershell
   .venv\Scripts\esphome compile esphome/buitenlamp.yaml
   ```

3. Upload over WiFi. Right after a `compile`, `upload` sends that new build:

   ```powershell
   .venv\Scripts\esphome upload esphome/buitenlamp.yaml --device kantoorlamp.local
   ```

4. **Smoke test after dark (T060).** The INFO log has no timestamps, so time this with a
   stopwatch:
   - wave once: the fade-in should take about 1 s;
   - the fade-out should start 60 s after Motion turns off in Home Assistant (allow up to 2 s
     for Home Assistant's delay) and take about 5 s;
   - switch the lamp on and off once from Home Assistant.
5. Fill in the rest of section 9 of quickstart.md:
   - the strip current at full brightness;
   - which scenarios passed, and any exceptions;
   - the date and the project version (`1.2.0`).
6. **Start the 60-day run (T061).** Write today's date in section 9. After 60 days without a
   restart, repeat V3 and V4 and record the result.

To mount the lamp outdoors, follow the "Outdoor installation" section of
[hardware.md](specs/001-pir-ledstrip-fade/contracts/hardware.md#outdoor-installation). A PIR
sensor cannot see through glass or clear plastic.

## Changing a setting later

The on period, the fade times, the time window and the reaction mode are changed in Home
Assistant (see "Settings in Home Assistant"). Only the other settings need an upload:

1. Edit the value in the `substitutions:` block. Each setting has a comment giving its unit
   and allowed range.
2. Build and upload over WiFi:

   ```powershell
   .venv\Scripts\esphome run esphome/buitenlamp.yaml --device kantoorlamp.local --no-states
   ```

   A value out of range stops the build with a message, and nothing is uploaded.

**Converting a 1.1.0 configuration**: 1.2.0 gives the timing defaults in seconds. Divide
`on_period_ms`, `fade_in_ms` and `fade_out_ms` by 1000 and enter them as `on_period_s`,
`fade_in_s` and `fade_out_s`.

After an upload, the lamp restarts off and in automatic mode, and ignores motion for 30 s.
