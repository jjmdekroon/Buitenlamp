# Feature Specification: Lamp Settings Adjustable from Home Assistant

**Feature Branch**: `002-ha-tunable-settings`

**Created**: 2026-11-09

**Status**: Draft

**Input**: User description: "Ik wil meer parameters kunnen instellen via home assistant. Deze zijn
de duur dat de lamp aan blijft nadat er niemand meer is gedetecteerd. De fadetijd van de lamp als
deze aan gaat en uit gaat. De tijdstippen dat de lamp reageerd op de sensor (bijvoorbeeld van 8 uur
ochtends tot 5 uur in de middag)"

(Translation: "I want to set more parameters from Home Assistant: how long the lamp stays on after
nobody is detected any more; the fade time of the lamp when it turns on and when it turns off; and
the times of day during which the lamp reacts to the sensor, for example from 8 in the morning to
5 in the afternoon.")

**Builds on**: [001-pir-ledstrip-fade](../001-pir-ledstrip-fade/spec.md). Everything in that
specification stays in force unless this specification changes it. Requirement ids of the form
"001/FR-0xx" refer to that specification.

## Clarifications

### Session 2026-11-09

- Q: How does the time window combine with the existing "only when dark" rule (001/FR-018)? The
  example 08:00–17:00 lies mostly in daylight. → A: The owner chooses it in Home Assistant with a
  reaction mode: "only when dark", "only in time window", "in time window and dark", or
  "always". The default is "only when dark", so the lamp behaves as before until the owner
  changes it.
- Q: How soon after a change in Home Assistant must a setting be safe from a power cut? → A: As
  soon as Home Assistant shows it: the setting is stored on the lamp within 2 seconds of the
  change, so a power cut from then on cannot lose it.
- Q: Should the brightness and colour chosen in Home Assistant (001/FR-031) get the same rule? →
  A: Yes: they are also stored within 2 seconds of the change.
- Q: Which exact text should Home Assistant show for the two reaction modes that use the time
  window? → A: "only in time window" and "in time window and dark", without "the", matching the
  status "In time window" (formerly written as "only in the time window" and "in the time window
  and dark").
- Q: May the last seconds of a very long fade-out show visible steps, given that the strip has
  only 256 brightness levels per colour? → A: Yes: at the end of a fade-out longer than
  10 seconds, the step between the lowest brightness levels may be visible; flicker is never
  allowed.
- Q: When the time becomes known and the conditions of the chosen mode are not met, does a
  motion counted during the unknown-time period end? → A: Yes, as for any end of the conditions
  (FR-012): the light fades out one on period later, also if the person keeps moving.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Owner changes how long the light stays on (Priority: P1)

The owner finds that the light goes out too soon (or stays on too long) after someone has left.
In Home Assistant, on the lamp's device page, they change the on period, for example from
1 minute to 3 minutes. From the next motion on, the lamp stays on for 3 minutes after the last
motion. No program update, cable or PC is needed, and the value is kept after a power cut.

**Why this priority**: The on period is the setting that most depends on where the lamp hangs and
how it is used; it is the main reason for wanting adjustment without reloading the program. On
its own this story already removes the need for a PC for the most common change.

**Independent Test**: In Home Assistant, set the on period to 2 minutes. Trigger the lamp once
(in the dark, or within the active time window) and step away; time how long it stays on after
the sensor stops reporting motion. Cut the power, restore it, and check that Home Assistant
still shows 2 minutes and the lamp still uses it.

**Acceptance Scenarios**:

1. **Given** the lamp appears in Home Assistant, **When** the owner opens its device page,
   **Then** the on period is shown with its current value and unit, and can be changed there.
2. **Given** the owner has set the on period to 2 minutes, **When** motion is detected and then
   ends, **Then** the light stays on until 2 minutes after the sensor stops reporting motion and
   then fades out.
3. **Given** the on period was changed in Home Assistant, **When** the power is cut and restored
   or the program is updated, **Then** the lamp still uses the changed value and Home Assistant
   shows it.
4. **Given** the owner enters a value outside the allowed range, **When** they try to save it,
   **Then** it is not accepted and the lamp keeps the previous value.

---

### User Story 2 - Owner changes the fade-in and fade-out times (Priority: P2)

The owner wants the light to come on faster, or go out more slowly. In Home Assistant they set
the fade-in time and the fade-out time separately. Every following fade (from motion and from
Home Assistant on/off commands) uses the new times.

**Why this priority**: Fade times are a matter of taste and are changed less often than the on
period, but the owner explicitly asked for them. The lamp works fine with the defaults.

**Independent Test**: Set the fade-in time to 3 seconds and the fade-out time to 10 seconds in
Home Assistant. Trigger the lamp and time the fade-in; let the on period end and time the
fade-out. Repeat with an on/off command from Home Assistant.

**Acceptance Scenarios**:

1. **Given** the fade-in time is set to 3 seconds, **When** motion turns the lamp on from off,
   **Then** the light reaches the on-setting 3 seconds after the detection.
2. **Given** the fade-out time is set to 10 seconds, **When** the on period ends, **Then** the
   light goes from the on-setting to completely off in 10 seconds.
3. **Given** changed fade times, **When** the owner switches the light on or off from Home
   Assistant, **Then** those fades use the changed times too.
4. **Given** the fade times were changed, **When** the power is cut and restored, **Then** the
   lamp still uses the changed times and Home Assistant shows them.

---

### User Story 3 - Owner chooses when the lamp reacts to motion (Priority: P2)

The owner wants the lamp to react to the sensor only during certain hours, for example from
08:00 to 17:00. In Home Assistant they set a start time and an end time, and choose a reaction
mode: "only when dark" (as before), "only in time window", "in time window and dark", or
"always". Outside the chosen conditions, motion does not switch the lamp on. The mode and the
window can be changed at any time and are kept after a power cut.

**Why this priority**: This changes when the lamp reacts at all, which is the biggest change in
behaviour in this feature. It is independent of Stories 1 and 2.

**Independent Test**: Choose "only in time window" and set the window so that it ends
5 minutes from now. Trigger the lamp before the end time (in daylight) and confirm it reacts;
wait until after the end time, let the light go out, then trigger the sensor again and confirm
the lamp stays off. Then choose each of the other modes and confirm the lamp reacts as described
for that mode.

**Acceptance Scenarios**:

1. **Given** mode "only in time window" and a window from 08:00 to 17:00, **When** motion is
   detected at 12:00 in daylight, **Then** the lamp turns on as in 001 Story 1.
2. **Given** mode "only in time window" and a window from 08:00 to 17:00, **When** motion is
   detected at 18:00, **Then** the strip stays off; the motion is still reported to Home
   Assistant.
3. **Given** mode "in time window and dark" and a window from 17:00 to 23:00, **When** motion
   is detected at 22:00 after sunset, **Then** the lamp reacts; at 23:30 (dark, outside the
   window) or at 17:30 before sunset (inside the window, light) it does not.
4. **Given** mode "always", **When** motion is detected at any time of day, **Then** the lamp
   reacts, whatever the window and the daylight.
5. **Given** mode "only when dark" (the default), **When** the feature is installed, **Then** the
   lamp reacts at the same moments as before this feature, whatever the window.
6. **Given** the light is on from motion, **When** the conditions of the chosen mode stop being
   met (for example the end time of the window passes), **Then** the running on period finishes
   and the light fades out normally; motion after that moment does not keep it on.
7. **Given** mode "only in time window" and a window from 22:00 to 06:00 (crossing
   midnight), **When** motion is detected at 02:00, **Then** the lamp reacts; at 12:00 it does
   not.
8. **Given** the conditions of the chosen mode are not met, **When** the owner switches the light
   on from Home Assistant, **Then** it turns on as before (manual mode is not limited by the
   reaction mode).

---

### Edge Cases

- **Setting changed while the light is on**: a running on period and a running fade finish with
  the value they started with; the new value applies from the next on period or fade. Nothing
  visibly jumps.
- **On period shortened while someone keeps moving**: the light stays on while motion continues;
  after the motion ends, the new on period applies.
- **Value outside the allowed range** (for example sent by an automation): it is refused; the
  lamp keeps and keeps showing the previous value.
- **Home Assistant unreachable**: the lamp keeps using the last values it received; they cannot be
  changed until Home Assistant is back.
- **Power cut or program update**: the lamp starts off and in automatic mode (001/FR-009,
  001/FR-030), with the settings last chosen in Home Assistant, also when the power is cut only a
  few seconds after a change (FR-004).
- **Default values after the program is updated with new defaults in the settings location**:
  values already chosen in Home Assistant are kept; the defaults only apply until a value has
  been chosen in Home Assistant.
- **Time unknown** (after power-up without WiFi or time source): the lamp cannot know whether it is
  dark or inside the time window, so in every mode it reacts to motion at any time until the time
  is known, as it already does for darkness (001/FR-023). When the time becomes known and the
  conditions are not met, this counts as the conditions ending (FR-012).
- **Start time equals end time**: the window covers the whole day, i.e. no time restriction. With
  mode "only in time window" the lamp then reacts at any time; with "in time window and dark" it
  reacts whenever it is dark.
- **Conditions start being met while someone is already moving** (window starts, or it becomes
  dark): the lamp turns on at the next motion detection after that moment (same as when it becomes
  dark, 001 edge cases).
- **Mode or window changed while the light is on from motion**: if the new conditions are still
  met, nothing changes; if they are no longer met, this counts as the conditions ending (FR-012):
  the running on period finishes and the light fades out normally.
- **Mode "only in time window" at sunrise**: sunrise does not end a running motion or on
  period; only the end of the window does.
- **Summer/winter time change**: the window follows the local clock time, so 08:00 stays 08:00
  local time after the change. Darkness keeps following the sun (001/FR-021).
- **Manual mode when the conditions end**: the light stays on until switched off from Home
  Assistant or until the manual-mode fallback (001/FR-029); the reaction mode only affects motion.
- **Manual-mode fallback while the conditions are not met**: when the lamp returns to automatic
  mode (001/FR-029) while the conditions of the chosen mode are not met, recent motion does not
  keep the light on; it fades out.

## Requirements *(mandatory)*

### Functional Requirements

**Adjustable from Home Assistant**

- **FR-001**: The owner MUST be able to see and change the following settings in Home Assistant,
  as part of the lamp's device, each labelled with what it controls and its unit:
  - the **on period** (how long the light stays on after the sensor stops reporting motion,
    001/FR-003), from 1 second to 24 hours in steps of 1 second, default 60 seconds;
  - the **fade-in time** (001/FR-002), from 0.1 to 10 seconds in steps of 0.1 second, default
    1 second;
  - the **fade-out time** (001/FR-005), from 0.1 to 60 seconds in steps of 0.1 second, default
    5 seconds;
  - the **time window start** and **time window end**, as a time of day with minute precision,
    default 00:00 and 00:00 (no restriction, FR-010);
  - the **reaction mode** (FR-011), one of "only when dark", "only in time window", "in time
    window and dark" and "always" (shown in Home Assistant exactly like this), default "only when
    dark".
- **FR-002**: A value outside its allowed range MUST NOT be accepted; the lamp MUST keep using and
  showing the previous value.
- **FR-003**: A changed value MUST be shown in Home Assistant within 2 seconds of the change and
  MUST be used from the next on period or fade that starts after the change. A running on period
  or fade MUST finish with the value it started with.
- **FR-004**: Every setting of FR-001 MUST be kept across power cuts, restarts and program
  updates, and MUST be stored only when it changes. A changed setting MUST be stored on the lamp
  within 2 seconds of the change, so a power cut from then on cannot lose it. The same 2-second
  rule applies to the brightness and colour chosen in Home Assistant (001/FR-031). After a
  restart, Home Assistant MUST show the kept values.
- **FR-005**: Until a value has been chosen in Home Assistant, the lamp MUST use the default from
  the settings location (001/FR-013). Once a value has been chosen in Home Assistant, it MUST take
  precedence over that default, also after a program update.
- **FR-006**: When Home Assistant is unreachable, the lamp MUST keep working with the last values
  it has, and MUST show its current values to Home Assistant when the connection returns.

**Timing**

- **FR-007**: The on period of 001/FR-003, 001/FR-004 and 001/FR-018 MUST be the on period set
  in FR-001 instead of a fixed 60 seconds (tolerance +1 s / −0 s).
- **FR-008**: The fade-in time and fade-out time of 001/FR-002, 001/FR-005, 001/FR-006,
  001/FR-017 and 001/FR-027 MUST be the times set in FR-001 instead of fixed 1 and 5 seconds
  (tolerance ±10 % or ±0.1 s, whichever is larger). Fades from a partial level keep using the
  same rate (a partial fade takes the matching part of the set time).
- **FR-009**: Fades MUST stay smooth (001/FR-007) at every allowed fade time, including the
  shortest and longest. At the end of a fade-out longer than 10 seconds, the step between the
  lowest brightness levels the strip can show MAY be visible; there MUST be no flicker.

**Time window**

- **FR-010**: The time window runs from the start time to the end time in local time (including
  summer and winter time). If the end time is earlier than the start time, the window crosses
  midnight (for example 22:00–06:00). If the start time equals the end time, there is no time
  restriction.
- **FR-011**: Once the time is known, motion MUST only switch the lamp on, or keep it on, when
  the conditions of the chosen reaction mode are met:
  - **only when dark**: it is dark (001/FR-018, 001/FR-019); the time window is not used. This is
    the behaviour before this feature;
  - **only in time window**: the current local time is inside the time window; daylight is not
    used;
  - **in time window and dark**: both hold;
  - **always**: no condition; motion always counts.

  When the conditions are not met, motion MUST be ignored and the strip MUST stay off. This
  replaces the darkness condition of 001/FR-018 with the chosen mode; the rest of 001/FR-018
  applies to the end of any condition through FR-012.
- **FR-012**: When the conditions of the chosen mode stop being met while the lamp is on from
  motion (the window ends, the dark period ends in a mode that uses darkness, or the owner changes
  the mode or window), the running on period MUST finish and fade out normally; motion after that
  moment MUST NOT extend it, and a motion still in progress at that moment counts as ended, so the
  on period starts then (same as the end of the dark period in 001/FR-018).
- **FR-013**: A motion that started while the conditions were not met MUST NOT count when they
  start being met; the lamp turns on at the next motion detection after that moment.
- **FR-014**: While the time is unknown, the lamp MUST react to motion at any time, in every
  reaction mode (extends 001/FR-023).
- **FR-015**: The reaction mode and time window MUST NOT affect manual mode: "on" and "off" from
  Home Assistant work at any time (001/FR-027). When the manual-mode fallback (001/FR-029) returns
  the lamp to automatic mode, recent motion keeps the light on only if the conditions of the
  chosen mode are met (or the time is unknown); otherwise the light MUST fade out. This replaces
  "if it is dark (or the time is unknown)" in 001/FR-029.
- **FR-016**: Motion MUST still be reported to Home Assistant when the conditions are not met
  (001/FR-025).
- **FR-017**: The lamp MUST report to Home Assistant whether it is currently inside the time
  window, within 2 seconds of a change, so the owner can see why the lamp does or does not react.
  The existing dark/daylight indicator (001/FR-024) stays.

### Key Entities

- **Adjustable settings**: on period, fade-in time, fade-out time, time window start, time
  window end and reaction mode, each with its allowed range or choices and its default. Chosen in
  Home Assistant, kept on the lamp across power cuts, stored only when changed.
- **Reaction mode**: which conditions decide whether motion counts: "only when dark", "only in
  time window", "in time window and dark" or "always".
- **Defaults**: the values from the settings location (001/FR-013), used until the owner has
  chosen a value in Home Assistant.
- **Time window**: a daily period in local time, possibly crossing midnight, or "no restriction"
  when start and end are equal; used by the reaction modes that include it (FR-011).
- **Window status**: whether the current local time is inside the time window, reported to Home
  Assistant; unknown while the time is unknown.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: The owner can change any of the six settings from Home Assistant in under
  1 minute, without a PC, cable or program update, and without opening the enclosure.
- **SC-002**: In 5 out of 5 trials per value, with the on period set to 10 seconds, 2 minutes and
  10 minutes, the light stays on until that time after the sensor stops reporting motion
  (tolerance +1 s / −0 s), then fades out.
- **SC-003**: In 5 out of 5 trials per value, with the fade-in time set to 0.5, 3 and 10 seconds
  and the fade-out time set to 0.5, 10 and 60 seconds, the measured fade time is within ±10 % or
  ±0.1 s (whichever is larger) of the set value, and an observer reports no flicker and no
  visible steps, apart from the lowest levels allowed by FR-009.
- **SC-004**: In 3 out of 3 power-cut trials, with the power cut 2 to 5 seconds after the last
  setting was changed, all six changed settings and the chosen brightness and colour are still
  shown in Home Assistant and used by the lamp after power returns.
- **SC-005**: For each of the four reaction modes, in 5 out of 5 trials motion while the mode's
  conditions are met turns the lamp on, and in 5 out of 5 trials motion while they are not met
  leaves the strip off (not applicable to "always").
- **SC-006**: In mode "only in time window", the lamp starts reacting within 1 minute after
  the window start time and stops starting new on periods within 1 minute after the window end
  time, on 3 different days and for a window that crosses midnight.
- **SC-007**: With the default settings, the lamp passes 001/SC-001, 001/SC-002 and 001/SC-009 to
  001/SC-011 unchanged: installing this feature changes nothing until the owner changes a value.
- **SC-008**: In 10 out of 10 trials, a value changed in Home Assistant is confirmed (shown as the
  lamp's current value) within 2 seconds.

## Assumptions

- The fade-in time and fade-out time are two separate settings, as they already are in the
  settings location; the owner's "fade time when it turns on and off" is read as both.
- The settings are changed by the owner in Home Assistant's own controls on the lamp's device
  page (and can therefore also be set by Home Assistant automations); no separate app or dashboard
  is part of this feature.
- Allowed ranges are the ranges already enforced for these values in the settings location
  (001 contracts/settings.md), so nothing that was valid before becomes invalid.
- The time window uses the local time zone of the installation, the same clock the lamp already
  synchronizes (001/FR-020); a minute's accuracy is enough.
- Assumption in 001 "there is no adjustment while the lamp is running" is replaced by this
  feature for the six settings above. The other tunables (stabilization period, manual-mode
  fallback time, number of LEDs, location, pins) stay build-time settings only.
- The on-setting (brightness and colour) remains adjustable from Home Assistant as in 001; this
  feature does not change how it is chosen or used, only how quickly it is stored (FR-004).
- A "reset to defaults" button is out of scope; the owner can enter the default values by hand.
- Only one time window per day is needed; separate windows per weekday are out of scope.
