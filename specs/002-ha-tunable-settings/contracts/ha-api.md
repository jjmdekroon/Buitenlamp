# Contract: Home Assistant Interface, additions in 1.2.0

**Feature**: [../spec.md](../spec.md) | **Covers**: FR-001 to FR-006, FR-017, SC-001, SC-008 |
**Background**: [../research.md](../research.md) S1–S5, S8, S11

This extends the 001 contract, [001 contracts/ha-api.md](../../001-pir-ledstrip-fade/contracts/ha-api.md).
Everything there stays valid. The device, discovery, encryption, existing entities and commands
do not change.

## Device

| Item | Value |
|------|-------|
| Project version | `esphome: project: version: "1.2.0"` (1.1.0 plus the entities below; nothing renamed) |

## New entities: settings (device page, section "Configuration")

All have `entity_category: config`, are stored on the lamp and keep their value after a power
cut, restart or update (FR-004). A change is confirmed in Home Assistant within 2 s (FR-003) and
stored on the lamp within 2 s (FR-004), so a power cut after that keeps it. The same holds for
the brightness and colour of the Buitenlamp light (001/FR-031).

| Entity (as shown) | Domain | Values | Default | Meaning |
|-------------------|--------|--------|---------|---------|
| On period | `number` (box) | 1–86400, step 1, unit `s` | 60 | time the light stays on after the sensor stops reporting motion |
| Fade-in time | `number` (box) | 0.1–10, step 0.1, unit `s` | 1.0 | time of a full fade from off to the on-setting; also used for brightness and colour changes from Home Assistant |
| Fade-out time | `number` (box) | 0.1–60, step 0.1, unit `s` | 5.0 | time of a full fade from the on-setting to off |
| Time window start | `time` | `HH:MM:SS`; seconds ignored | `00:00:00` | first minute of the window |
| Time window end | `time` | `HH:MM:SS`; seconds ignored | `00:00:00` | first minute **after** the window. Equal to start: no restriction. Earlier than start: the window crosses midnight |
| Reaction mode | `select` | `only when dark`, `only in time window`, `in time window and dark`, `always` | `only when dark` | which conditions let motion switch the lamp on |

Entity ids in Home Assistant are formed from the device name and the entity name as usual
(for example `number.kantoorlamp_on_period`); the exact ids are recorded in the quickstart
results.

## New entity: status

| Entity (as shown) | Domain | Device class / unit | States | Meaning |
|-------------------|--------|---------------------|--------|---------|
| In time window | `binary_sensor` | none | `on` / `off` / `unknown` | whether the current local time is inside the window; `unknown` while the time is unknown. Shown in every reaction mode |

## Commands

| Home Assistant action | Lamp behaviour | Latency |
|-----------------------|----------------|---------|
| `number.set_value` on On period, Fade-in time or Fade-out time | value in range: stored and used from the next on period or fade (a running one keeps its length). Out of range: refused, a warning in the lamp's log, the old value stays (FR-002) | confirmed within 2 s |
| `time.set_value` on Time window start or end | stored; In time window and the reaction conditions are re-evaluated at once. If motion is no longer allowed, a running on period finishes and the light fades out normally (FR-012) | confirmed within 2 s |
| `select.select_option` on Reaction mode | as for the window. An unknown option is refused by Home Assistant | confirmed within 2 s |

The settings cannot be changed while Home Assistant is unreachable; the lamp keeps using its
stored values (FR-006).

## Compatibility rules (additions)

- The six setting entities' **names** are part of the contract: the lamp's storage key is
  derived from them, so renaming one loses its stored value (research S8).
- The **order of the Reaction mode options** is part of the contract: the lamp stores the
  index of the chosen option. New options may only be appended (research S3).
- **Version 1.2.0 (2026-11-09)**: adds the seven entities above. No existing entity changes;
  dashboards and automations from 1.1.0 keep working.
