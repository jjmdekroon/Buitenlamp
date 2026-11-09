# Contract: Owner Settings

**Feature**: [../spec.md](../spec.md) | **Covers**: FR-013, FR-022, FR-030, User Story 5,
SC-008

The owner changes the lamp's behaviour only by editing these two places and uploading the
program again over WiFi. Nothing else in the configuration needs to change.

- **Settings**: the `substitutions:` block at the top of `esphome/buitenlamp.yaml`.
- **Secrets**: `esphome/secrets.yaml`.

Every value has a comment giving its unit. Numeric ranges are enforced with `static_assert` in
one `on_boot` lambda, so a value out of range fails the build instead of producing a lamp that
misbehaves. ESPHome itself validates pins, the channel order, the coordinates and the log level.

## `substitutions:`: owner settings (committed)

| Name | Default | Allowed range | Unit | Spec |
|------|---------|---------------|------|------|
| `fade_in_ms` | `"1000"` | 100–10000 | ms | FR-002 |
| `on_period_ms` | `"60000"` | 1000–86400000 | ms | FR-003 |
| `fade_out_ms` | `"5000"` | 100–60000 | ms | FR-005 |
| `pir_stabilization_ms` | `"30000"` | 0–120000 | ms | FR-010 |
| `pir_min_pulse_ms` | `"50"` | 1–1000 | ms | FR-011 |
| `manual_fallback_ms` | `"600000"` | 60000–86400000 | ms | FR-029 |
| `num_leds` | `"60"` | 1–240 (research R3) | LEDs | FR-013 |
| `led_channel_order` | `"GRBW"` | `GRBW`, `RGBW`, `WRGB` or any order of R, G, B, W (checked by ESPHome) | — | strip type |
| `on_red`, `on_green`, `on_blue`, `on_white` | `"0.0"`, `"0.0"`, `"0.0"`, `"1.0"` | 0.0–1.0 each, not all 0 | fraction of full | FR-017 (default on-colour, used until a colour is chosen in Home Assistant) |
| `on_brightness` | `"1.0"` | 0.01–1.0 | fraction of full | FR-017 (default brightness, used until a brightness is chosen in Home Assistant) |
| `latitude` | example value; **must be set** | −90 to 90 | degrees, north positive | FR-019 |
| `longitude` | example value; **must be set** | −180 to 180 | degrees, east positive | FR-019 |
| `pin_pir` | `"GPIO4"` | a valid ESP32-C3 GPIO, not 2, 8, 9 or 11–19 | — | [hardware.md](hardware.md) |
| `pin_led_data` | `"GPIO5"` | same rule as `pin_pir`, and ≠ `pin_pir` | — | [hardware.md](hardware.md) |
| `ntp_server_1`, `ntp_server_2` | `"pool.ntp.org"`, `"time.cloudflare.com"` | host names | — | FR-020 |
| `log_level` | `"INFO"` | `INFO` (delivered), `DEBUG` (validation only) | — | constitution, Logging |
| `device_name` | `"buitenlamp"` | lowercase letters, digits, `-` | — | FR-024 |
| `friendly_name` | `"Buitenlamp"` | ≤ 32 characters | — | FR-024 |

The `on_*` values are only **defaults**. Once a brightness or colour is chosen in Home
Assistant, the lamp stores it and keeps using it after restarts and updates (FR-031).
Changing an `on_*` value afterwards therefore has no visible effect; choose the brightness or
colour in Home Assistant instead.

## `substitutions:`: fixed constants (not owner settings)

These sit in a separate, clearly labelled section of the same block, so there are no magic
numbers in the logic (constitution, Principle V). Changing them changes behaviour that the spec
fixes.

| Name | Value | Why fixed |
|------|-------|-----------|
| `sun_horizon_deg` | `"-0.83333"` | standard sunrise/sunset elevation. The spec allows no offset (FR-019) |
| `led_gamma` | `"2.8"` | perceptual fade curve (FR-007, research R6) |
| `led_max_refresh` | `"10ms"` | at most 100 frames per second (research R3) |
| `daylight_check_interval` | `"10s"` | how often the sun elevation is recomputed (SC-010) |
| `brightness_report_interval` | `"1s"` | at most one brightness report per second (FR-025) |
| `time_retry_interval` | `"60s"` | FR-020: retry at least once a minute while the time is unknown |
| `strip_level` | `"(id(strip).current_values.get_state() * id(strip).current_values.get_brightness())"` | a shared expression, not a number: the strip's current level, 0–1. The one definition used by both fade scripts and the Brightness sensor (constitution IV.2) |

## `secrets.yaml`: private settings (never committed; template `secrets.example.yaml`)

| Name | Example | Notes |
|------|---------|-------|
| `wifi_ssid` | `"MyHomeWiFi"` | 2.4 GHz network |
| `wifi_password` | `"…"` | |
| `api_encryption_key` | 44 base64 characters | 32 random bytes. It protects the Home Assistant connection **and** updates over WiFi: it is the spec's "update password" (FR-030, research R12) |

Generate the key once on the PC:

```powershell
python -c "import secrets,base64;print(base64.b64encode(secrets.token_bytes(32)).decode())"
```

If `secrets.yaml` is missing, `esphome config` stops and names the missing secret. Copy
`secrets.example.yaml` to `secrets.yaml` and fill it in.
