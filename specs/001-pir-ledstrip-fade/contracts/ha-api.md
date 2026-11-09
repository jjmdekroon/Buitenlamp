# Contract: Home Assistant Interface (ESPHome native API)

**Feature**: [../spec.md](../spec.md) | **Covers**: FR-024 to FR-029, SC-012 to SC-015 |
**Background**: [../research.md](../research.md) R8, R9, R10

This is what the lamp offers Home Assistant through its built-in ESPHome integration. Anything
not listed here is not part of the contract. This replaces the earlier MQTT contract.

## Device

| Item | Value |
|------|-------|
| ESPHome `name` (host name) | `${device_name}`, default `buitenlamp` → `buitenlamp.local` |
| Friendly name (device name in Home Assistant) | `${friendly_name}`, default `Buitenlamp` |
| Project (shown as model and firmware version) | `esphome: project: name: diy.buitenlamp`, `version: "1.1.0"` (1.0.0 had a switch instead of the light; see Compatibility rules) |
| Discovery | mDNS service `_esphomelib._tcp`, advertised automatically |
| Port | 6053 (ESPHome default) |
| Encryption | Noise, with the 32-byte base64 key `api_encryption_key` from `secrets.yaml` |

**Adding the device** (FR-024, SC-012): Home Assistant shows "Discovered: Buitenlamp". The owner
clicks **Add** and pastes the encryption key. No address is typed.

## Entities

| Entity (as shown) | Domain | Device class / unit | States | Meaning |
|-------------------|--------|---------------------|--------|---------|
| Buitenlamp | `light` | colour mode RGBW, with brightness | `on` / `off`; attributes `brightness`, `rgbw_color` | `on` while the light fades in or is on; `off` while it fades out or is off. Brightness and colour show the on-setting (FR-017), the values every "on" uses |
| Motion | `binary_sensor` | `motion` | `on` / `off` | filtered motion. Reported in daylight and in manual mode too, but not during the stabilization period after boot |
| Daylight | `binary_sensor` | `light` | `on` (light) / `off` (dark) / `unknown` | `unknown` until the time is known after power-up |
| Brightness | `sensor` | `%`, state class `measurement`, 0 decimals | 0–100 | perceived brightness. Updated at most once a second, and only when it changes |
| Mode | `sensor` (ESPHome text sensor) | — | `automatic` / `manual` | `manual` after an "on" from Home Assistant |

**Not exposed**: the raw PIR input and the sun elevation. The light has no effects.

## Commands

| Home Assistant action | Lamp behaviour ([data-model](../data-model.md) T7/T9) | Latency |
|-----------------------|----------------------------------------------------------|---------|
| `light.turn_on` (no brightness or colour) | manual mode; fade to the on-setting at the normal rate; stay on, day or night, ignoring motion and the on period. Works the same when the light is already `on` (for example after motion) | fade starts within 1 s |
| `light.turn_on` with `brightness` and/or `rgbw_color` | the values become the new on-setting, kept after a power cut (FR-031); otherwise as above: manual mode, and the light fades to them | fade starts within 1 s |
| `light.turn_off` | automatic mode; fade out at the normal rate. A motion that is still going on needs to end before the next detection can switch the light on | fade starts within 1 s |
| `light.toggle` | `on` → turn_off, `off` → turn_on | within 1 s |

A `transition` sent with a command is ignored: fades always use `fade_in_ms` and `fade_out_ms`
(FR-027). `flash` and `effect` are not supported.

## Availability

| Event | What Home Assistant shows |
|-------|---------------------------|
| Lamp loses power or network silently | unavailable after 90–110 s (client ping every 20 s, 90 s timeout); spec limit 2 min (FR-026) |
| Lamp restarts (update, watchdog) | unavailable at once (the connection drops), then available again after the reconnect |
| Home Assistant restarts | the lamp sees the client disconnect: the manual-fallback timer starts. On reconnect the timer stops and Home Assistant reads every current state (FR-028) |

## Time service

The lamp also takes the time from Home Assistant over the same connection (`time: platform:
homeassistant`), next to SNTP (research R8). Home Assistant needs no setup for this.

## Reachability rule (FR-029)

Home Assistant counts as reachable while at least one API client that subscribes to entity
states is connected. `esphome logs` also subscribes by default; use
`esphome logs … --no-states` so that watching the log does not keep the lamp in manual mode.

## Compatibility rules

- The entity names, object ids (`buitenlamp`, `motion`, `daylight`, `brightness`, `mode`) and
  state values above are a contract. Renaming them breaks Home Assistant history, dashboards
  and automations. A change needs a new `project: version` and a note in the quickstart.
- New entities may be added without breaking anything.
- **Version 1.1.0 (2026-11-02)**: `switch.buitenlamp` is replaced by `light.buitenlamp` (same
  name and object id, other domain). After updating, Home Assistant shows the old switch as
  unavailable; delete it on the device page, and change automations and dashboards from
  `switch.turn_on/turn_off` to `light.turn_on/turn_off`.
