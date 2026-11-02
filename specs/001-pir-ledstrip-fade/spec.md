# Feature Specification: PIR-Controlled LED Strip with Fade

**Feature Branch**: `001-pir-ledstrip-fade`

**Created**: 2026-11-01

**Status**: Draft (updated 2026-11-02: brightness and colour from Home Assistant)

**Input**: User description: "I have a ledstrip i need to switch on and off using a pir. The on/off switching must be a fade so the light is faded to on and slowly faded to off. Fade on is in 1 sec. The light is minimal 1 minute always on and then fades off in 5 sec."

**Change request (2026-11-02)**: "Is it possible to also set the brightness and colour of the
lamp in Home Assistant?"

## Clarifications

### Session 2026-11-01

- Q: What kind of LED strip are you connecting to the Arduino? → A: Addressable strip (3 wires:
  5 V, data, GND; e.g., WS2812B, NeoPixel, SK6812), each LED individually controllable.
- Q: What colour should the strip light up when it is on? → A: Bright white. The strip is
  RGBW, so "on" uses the dedicated white channel at full; red, green, and blue stay off.
- Q: Should the lamp react to motion during the day too, or only when it's dark? → A: Only
  when there is no daylight. The board is an ESP32-C3; it gets the current time over WiFi and
  works out from the sun's position at the installation location whether it is dark (no
  light sensor).
- Q: What should the lamp do when it detects motion but doesn't know the time yet, for
  example after a power cut while WiFi or the internet is down? → A: React to motion day and
  night until the time is known.
- Q: Should "dark" start exactly at sunset and end at sunrise, or only once it's actually
  dark? → A: Exactly sunset to sunrise, fixed (no twilight margin, no offset setting).
- Q: Should the lamp also work with home automation? → A: Yes. A home automation system
  running on a separate Raspberry Pi must be able to discover the lamp, receive its status,
  and switch the light on and off.
- Q: Which home automation software runs on your Raspberry Pi? → A: Home Assistant.
- Q: When you switch the light on or off from Home Assistant, how should that work with the
  motion sensor and the day/night rule? → A: Manual override until switched off: "on" fades
  the light on and keeps it on, day or night, ignoring motion and the 1-minute timer; "off"
  fades it out and returns the lamp to reacting to motion in the dark.
- Q: The light was switched on from Home Assistant, and then the Raspberry Pi or the WiFi
  stops working, so no "off" can arrive. What should the lamp do? → A: Go back to automatic
  mode once Home Assistant has been unreachable for 10 minutes; the light fades out unless
  there is motion in the dark.
- Q: How is the lamp's program built, and how does it talk to Home Assistant? → A: With
  ESPHome instead of a hand-written Arduino program. It is built and loaded from the owner's
  PC, and Home Assistant talks to it through its built-in ESPHome integration, so no message
  broker is needed.
- Q: When the lamp first shows up in Home Assistant, is it acceptable to click "Add" once and
  paste the lamp's encryption key once? → A: Yes. Home Assistant finds the lamp by itself;
  the owner adds it with one confirmation and the encryption key, without entering an
  address, and the connection is encrypted.
- Q: Should you be able to load a new version of the program onto the lamp over WiFi, once it
  is mounted outside? → A: Yes, over WiFi, protected by a password kept with the other
  secrets; only the first load needs a USB cable. The lamp restarts after an update and
  starts off.
- Q: Where should the lamp get the current time from: an internet time service, Home
  Assistant, or both? → A: Both, whichever answers first; each keeps the clock synchronized
  while it is reachable, so the time is only unknown when both are unreachable.
- Q: Home Assistant's ESPHome connection needs 90–110 s to notice that a device lost power.
  Should the offline limit change from 60 s? → A: Yes, Home Assistant shows the lamp offline
  within 2 minutes.
- Q: May the lamp's on/off control appear in Home Assistant as a switch (with a lightbulb
  icon) instead of a light, so commands from Home Assistant are always told apart from
  motion? → A: Yes, a switch named after the lamp. *(Superseded on 2026-11-02: the lamp now
  appears as a light, see below.)*
- Q: Does the 1-minute on period count from the start or the end of a motion detection? → A:
  From the end: the light stays on until 1 minute after the sensor stops reporting motion, so
  the sensor's own hold time adds to it. Counting from the start would let the timer run out
  while someone keeps moving and the sensor stays on.
- Q: Someone keeps moving in front of the sensor while the sun rises: should the light stay on
  until they stop? → A: No. At sunrise the ongoing motion stops counting; the light stays on
  for one more on period and then fades out, even if they keep moving.

### Session 2026-11-02

- Q: How should brightness and colour be set from Home Assistant? → A: The lamp appears in
  Home Assistant as one light named after the lamp, instead of a switch, with on/off, a
  brightness slider and a colour control (red, green, blue and white) on one card. On and off
  work as before: "on" means manual mode until "off".
- Q: Which light do the chosen brightness and colour apply to? → A: To every "on", both from
  motion and from Home Assistant. They replace the fixed on-colour; the values in the settings
  location are only the defaults until a brightness and colour are chosen.
- Q: Should the chosen brightness and colour be kept after a power cut or restart? → A: Yes.
  They are stored on the lamp, and only when they change. The lamp still starts off and in
  automatic mode.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Light fades on when someone approaches and fades off afterwards (Priority: P1)

A person walks into the area watched by the motion sensor. The LED strip smoothly fades from
off to full brightness in 1 second. It stays on for 1 minute, then slowly fades to off over
5 seconds. The person never sees the light snap on or off.

**Why this priority**: This is the whole purpose of the lamp. Without it, nothing else in this
feature has value. On its own it already gives a working motion-activated outdoor light.

**Independent Test**: After dark, power the device with the sensor and LED strip connected.
Wave a hand in front of the sensor once, then step out of view. Time the fade-in, the on
period, and the fade-out with a stopwatch.

**Acceptance Scenarios**:

1. **Given** the lamp is off, it is dark, and the sensor is ready, **When** the sensor
   detects motion, **Then** the strip fades from off to full brightness, reaching full
   brightness 1 second after the detection.
2. **Given** the lamp has just reached full brightness after a single detection, **When** no
   further motion is detected, **Then** the strip stays at full brightness until 1 minute
   after the sensor stops reporting that motion.
3. **Given** the 1-minute on period has ended without new motion, **When** the fade-out
   starts, **Then** the strip goes from full brightness to completely off in 5 seconds.
4. **Given** the fade-out has completed, **When** motion is detected again, **Then** the full
   cycle (fade-in, on period, fade-out) runs again.

---

### User Story 2 - Light stays on while someone is still there (Priority: P2)

Someone stays in the area, for example while unloading a car or searching for keys. Every
time the sensor detects movement, the 1-minute on period starts again, so the light does not
fade out on them. If the light has already started fading out and the person moves, it fades
back up to full brightness instead of going dark.

**Why this priority**: Without this, a person who stays longer than 1 minute is left in the
dark. It improves on Story 1 but needs Story 1 to exist first.

**Independent Test**: Trigger the lamp, then keep moving in front of the sensor every
30 seconds for 3 minutes; confirm the light never dims. Then let the fade-out start, move in
front of the sensor halfway through, and confirm the light smoothly returns to full
brightness.

**Acceptance Scenarios**:

1. **Given** the lamp is at full brightness, **When** motion is detected again before the
   on period ends, **Then** the on period restarts, so the light stays on until 1 minute after
   the most recent motion ends.
2. **Given** the lamp is fading in, **When** motion is detected again, **Then** the fade-in
   continues without interruption and the 1-minute on period counts from the most recent
   detection.
3. **Given** the lamp is fading out, **When** motion is detected, **Then** the fade-out stops
   and the light smoothly rises from its current brightness back to full brightness, with no
   jump or flash, and a new 1-minute on period starts.
4. **Given** someone keeps moving in the area continuously, **When** time passes, **Then** the
   light stays at full brightness for as long as motion keeps being detected while it is dark.

---

### User Story 3 - Lamp only reacts when it is dark (Priority: P2)

During the day nobody needs the light, so motion is ignored and the strip stays off. Once it
is dark at the installation location, motion turns the lamp on as in Story 1. The lamp knows
whether it is dark from the current time, which it gets over WiFi, and the position of the
sun at the configured location; no light sensor is needed.

**Why this priority**: The owner explicitly wants the lamp off in daylight. It saves power and
avoids pointless switching, but the lamp is still useful without it, so it ranks after the
core on/off behavior.

**Independent Test**: During the day, walk in front of the sensor and confirm the strip stays
off. Then set the location in the settings to a place where it is currently night, load the
program, and confirm motion now turns the lamp on.

**Acceptance Scenarios**:

1. **Given** it is daylight at the installation location, **When** the sensor detects motion,
   **Then** the strip stays off.
2. **Given** it is dark at the installation location, **When** the sensor detects motion,
   **Then** the lamp behaves as in Story 1.
3. **Given** the lamp is on, **When** it becomes light (the dark period ends), **Then** the
   running on period finishes and the lamp fades out normally; further motion does not turn
   it on or keep it on.
4. **Given** the clocks change between summer and winter time, **When** the next evening
   comes, **Then** the dark period still starts at the correct moment.
5. **Given** the lamp has just been powered up and could not get the time (WiFi down, or both
   the internet and Home Assistant unreachable), **When** the sensor detects motion, **Then** the lamp turns on as in Story 1,
   whether it is day or night, until the time becomes known.

---

### User Story 4 - Home automation sees and controls the lamp (Priority: P3)

The owner runs Home Assistant on a Raspberry Pi in the same home network. The lamp shows up
there by itself as one device, without typing in an address. From Home Assistant, the owner
can see whether the light is on, whether motion was detected, and whether the lamp is online,
can switch the light on and off, and can choose its brightness and colour. The chosen
brightness and colour are used every time the lamp turns on, by motion too, and are kept after
a power cut.

**Why this priority**: The owner wants the lamp to be part of their home automation, but the
lamp already does its main job without it, so it comes after the core behavior.

**Independent Test**: Power up the lamp with the home automation running. Confirm the lamp
appears in the home automation as a discovered device and can be added with one confirmation
and its encryption key, that its status changes when someone walks past, and that switching it
on and off from the home automation works. Then choose a lower brightness and another colour,
switch the lamp off, and confirm that the next motion in the dark turns it on in that
brightness and colour, also after a power cut.

**Acceptance Scenarios**:

1. **Given** the lamp is powered up on the home network, **When** the home automation is
   running, **Then** Home Assistant shows the lamp as a newly discovered device by itself, and
   the owner adds it with one confirmation and the lamp's encryption key, without entering an
   address.
2. **Given** the lamp appears in the home automation, **When** the light turns on or off or
   motion is detected, **Then** the new status is shown in the home automation within
   2 seconds.
3. **Given** the lamp loses power, **When** the home automation is checked, **Then** it shows
   the lamp as offline within 2 minutes.
4. **Given** the home automation is switched off or unreachable, **When** someone walks past
   in the dark, **Then** the lamp still works as in Stories 1–3.
5. **Given** the light is off, day or night, **When** the owner switches it on from Home
   Assistant, **Then** it fades on in 1 second and stays at full brightness, without the
   1-minute timer, until it is switched off from Home Assistant.
6. **Given** the light was switched on from Home Assistant, **When** motion is detected,
   **Then** the light stays on unchanged; the motion is still reported.
7. **Given** the light is on (switched on from Home Assistant or by motion), **When** the
   owner switches it off from Home Assistant, **Then** it fades out over 5 seconds and the
   lamp goes back to reacting to motion in the dark.
8. **Given** the light was switched on from Home Assistant, **When** Home Assistant has been
   unreachable for 10 minutes, **Then** the lamp returns to automatic mode and the light fades
   out, unless it is dark and someone moved within the last on period.
9. **Given** the lamp appears in Home Assistant, **When** the owner chooses a brightness or
   colour there, **Then** the light fades to it within about 1 second, the lamp is in manual
   mode (as after "on"), and from then on every "on", from motion or from Home Assistant, uses
   that brightness and colour.
10. **Given** a brightness and colour were chosen in Home Assistant, **When** the power is cut
    and restored or the program is updated, **Then** the lamp starts off and in automatic mode,
    and the next "on" uses the chosen brightness and colour.

---

### User Story 5 - Owner adjusts the timing (Priority: P4)

The owner finds that 1 minute is too short for their driveway, or wants a slower fade-out.
They change the fade-in time, on period, or fade-out time in one clearly labelled settings
location, load the updated program onto the device over WiFi, without taking it down or
opening its enclosure, and the lamp behaves with the new values.

**Why this priority**: The requested defaults work out of the box; tuning is a convenience
that becomes useful after the lamp has been installed and used for a while.

**Independent Test**: Change the on period from 1 minute to 2 minutes in the settings
location, load the program, trigger the lamp once, and confirm it now stays on for
2 minutes before fading out.

**Acceptance Scenarios**:

1. **Given** the default settings, **When** the owner changes any one of the fade-in time,
   on period, or fade-out time and loads the program, **Then** the lamp uses the new value
   and every other behavior stays the same.
2. **Given** the settings location, **When** the owner reads it, **Then** each value is
   labelled with what it controls and its unit (seconds or milliseconds).

---

### Edge Cases

- **Motion during fade-in**: the fade-in completes normally; the on period counts from the
  latest detection.
- **Motion during fade-out**: the light returns smoothly to full brightness from wherever it
  is, without first dropping to off and without a visible jump.
- **Continuous motion for a long time** (e.g., someone working outside for an hour): the light
  stays at full brightness the whole time while it is dark, with no flicker or dimming.
- **Power cut while the light is on**: when power returns, the lamp starts off and behaves as
  after a normal power-up.
- **LEDs showing random colours at power-up**: addressable LEDs can light up briefly in random
  colours before they receive their first instruction; the lamp clears them so no flash is
  visible.
- **Sensor warming up after power-up**: motion sensors of this type can give false signals
  for a short time after power is applied; detections during this stabilization period are
  ignored so the lamp does not switch on by itself after a power cut.
- **Electrical glitch on the sensor line**: a signal shorter than 50 milliseconds is not
  treated as motion.
- **Sensor disconnected**: the lamp stays off; a loose or missing sensor wire does not make
  the light switch on at random.
- **Sensor stuck reporting motion**: the lamp stays on, because this cannot be told apart from
  continuous presence (accepted limitation, see Assumptions).
- **Months of continuous operation**: timing stays correct no matter how long the device has
  been running without a restart.
- **Dark period ends while the lamp is on**: the current on period runs to its end and the
  lamp fades out normally; it does not cut off abruptly at sunrise. If someone is still moving
  at that moment, the on period starts at sunrise, so the light still goes out even if they keep
  moving.
- **It becomes dark while someone is already moving in the area**: the lamp turns on at the
  next motion detection after the dark period starts.
- **WiFi unavailable at power-up, or both the internet and Home Assistant unreachable**: the
  time is unknown, so the lamp reacts to motion day and night; as soon as the time is obtained
  from either source, it switches to following the dark period. A cycle already running at
  that moment finishes normally.
- **Internet down but Home Assistant reachable at power-up** (or the other way round): the
  lamp gets the time from the source that answers and follows the dark period as usual.
- **WiFi or internet lost after the time is known**: the lamp keeps time on its own and keeps
  following the dark period; it tries to re-synchronize until it succeeds.
- **Summer/winter time change**: has no effect on when the dark period starts or ends, because
  darkness follows the sun, not the local clock.
- **Home automation unreachable**: the lamp keeps working on motion and darkness alone,
  reconnects automatically when the home automation comes back, and then reports its current
  status.
- **"On" from Home Assistant while the light is already on from motion**: there is no visible
  change; the light now stays on until switched off from Home Assistant.
- **"Off" from Home Assistant while someone is moving in the dark**: the light fades out; the
  next motion detection turns it on again as usual.
- **"Off" from Home Assistant while the light is already off**: nothing happens.
- **Brightness or colour changed from Home Assistant while the light is on from motion**: the
  light fades to the new brightness and colour, and, as with every "on" from Home Assistant,
  the lamp is now in manual mode until switched off.
- **Brightness or colour chosen while the light is off**: Home Assistant switches the light on
  in that brightness and colour (manual mode); after "off", motion uses them.
- **Home Assistant asks for a specific fade time**: it is ignored; the lamp always fades at the
  rates set in the settings location.
- **Power cut while the light is switched on from Home Assistant**: when power returns, the
  lamp starts off and reacts to motion as usual; Home Assistant shows it as off.
- **Home Assistant or WiFi goes away while the light is switched on from Home Assistant**:
  after 10 minutes without contact, the lamp returns to automatic mode and the light fades
  out, unless it is dark and someone moved within the last on period. A short interruption
  (under 10 minutes) changes nothing.
- **Program update over WiFi while the light is on**: the lamp restarts to apply the update
  and then behaves as after a normal power-up: it starts off, in automatic mode, and ignores
  motion during the stabilization period.
- **Update attempt without the right password**: the update is refused and the lamp keeps
  running its current program without interruption.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The system MUST detect motion using a PIR motion sensor.
- **FR-002**: When motion is detected while the lamp is off, the system MUST raise the LED
  strip from off to full brightness over 1 second (tolerance ±0.1 s).
- **FR-003**: The system MUST keep the strip at full brightness until 60 seconds after the
  motion sensor stops reporting the most recent motion (tolerance +1 s / −0 s). The sensor's
  own hold time therefore adds to the on period.
- **FR-004**: Each motion detection while the lamp is fading in or at full brightness MUST
  restart the 60-second on period.
- **FR-005**: When the on period ends without a new detection, the system MUST lower the
  strip from full brightness to completely off over 5 seconds (tolerance ±0.25 s).
- **FR-006**: When motion is detected during the fade-out, the system MUST stop the fade-out,
  raise the brightness from its current level to full at the same rate as the fade-in
  (so it reaches full in 1 second or less), and start a new 60-second on period.
- **FR-007**: All brightness changes MUST look smooth to the eye: no visible steps, flicker,
  or sudden jumps during fade-in, at full brightness, or during fade-out. The perceived
  brightness MUST change evenly over the fade, not mostly at the start or end.
- **FR-008**: When the fade-out completes, the strip MUST be completely off, with no residual
  glow.
- **FR-009**: When power is applied or restored, the lamp MUST start off, with no visible flash
  or random colours on any LED.
- **FR-010**: The system MUST ignore motion detections during a sensor stabilization period
  after power-up (default: 30 seconds).
- **FR-011**: Sensor signals shorter than 50 milliseconds MUST NOT count as motion.
- **FR-012**: If the motion sensor is disconnected, the lamp MUST stay off.
- **FR-013**: The fade-in time, on period, fade-out time, stabilization period, manual-mode
  fallback time, number of LEDs on the strip, default on-colour and brightness, and
  installation location (latitude and longitude) MUST each be adjustable in a single, clearly
  labelled settings location, without changing any other part of the program.
- **FR-014**: The lamp MUST operate without a display or computer connected. WiFi is used to
  obtain the current time and to talk to the home automation; the lamp MUST keep responding
  to motion when WiFi, the internet, or the home automation is unavailable.
- **FR-015**: Timing behavior MUST remain correct during uninterrupted operation of any
  length, including beyond 60 days.
- **FR-016**: All LEDs on the strip MUST show the same colour and brightness at the same
  moment, so the whole strip fades as one; no running, chasing, or per-LED effects.
- **FR-017**: "Full brightness" MUST mean the on-setting: the brightness and colour (red,
  green, blue and white levels) last chosen in Home Assistant (FR-027), or, until one has been
  chosen, the defaults from the settings location (the white channel at its maximum, red,
  green and blue off, brightness 100 %). A fade from off to the on-setting MUST take the
  fade-in time and a fade from the on-setting to off the fade-out time, whatever the chosen
  brightness. Fades MUST scale all channels together, so the colour stays the same while only
  the brightness changes.
- **FR-018**: Once the time is known, the lamp MUST only turn on in response to motion while it
  is dark at the installation location. During daylight, motion MUST be ignored and the strip
  MUST stay off. If the dark period ends while the lamp is on, the running on period MUST
  finish and fade out normally, and motion after the dark period ends MUST NOT extend it. A
  motion still in progress at that moment counts as ended, so the on period starts then.
- **FR-019**: Whether it is dark MUST be determined from the current date and time and the
  sun's position at the installation location, without a light sensor. "Dark" means the
  period from sunset to sunrise, matching the published sunset and sunrise times for the
  location, with no offset or twilight margin.
- **FR-020**: The system MUST obtain the current time over WiFi from two independent sources,
  a public internet time service and Home Assistant, using whichever answers first. It MUST
  obtain the time within 60 seconds of power-up when either source is reachable, retry at
  least once a minute while the time is unknown, and re-synchronize at least once every
  24 hours while either source is reachable. Between synchronizations, its clock MUST stay
  within 1 minute of the correct time.
- **FR-021**: The dark period MUST follow the sun, so changes between summer and winter time
  MUST NOT shift it.
- **FR-022**: The WiFi network name and password, the encryption key for the home automation
  connection, and the update password (FR-030) MUST be kept apart from the other settings, so
  the program and its settings can be shared without revealing them.
- **FR-023**: While the time is unknown (not yet obtained since power-up), the lamp MUST react
  to motion day and night. Once the time is obtained, FR-018 applies; a cycle already running
  at that moment MUST finish normally.
- **FR-024**: The lamp MUST make itself discoverable on the home network, so Home Assistant
  finds it without the owner entering an address. Adding it MUST take no more than one
  confirmation and entering the lamp's encryption key once, and the connection between the
  lamp and Home Assistant MUST be encrypted. It MUST appear as one device containing a light
  named after the lamp (with on/off, brightness and colour control), a motion sensor, and a
  dark/daylight indicator.
- **FR-025**: The lamp MUST report its status to the home automation within 2 seconds of any
  change: whether the light is on or off, its current brightness, whether motion is detected,
  whether it is currently dark, and whether it is in automatic or manual mode.
- **FR-026**: The home automation MUST be able to see whether the lamp is online; when the lamp
  loses power or network, the home automation MUST show it as offline within 2 minutes.
- **FR-027**: The home automation MUST be able to switch the light on and off and choose its
  brightness and colour; the lamp MUST start acting on a command within 1 second.
  - "On" MUST put the lamp in manual mode: the light fades from its current brightness to
    the on-setting (FR-017) at the normal fade-in rate and stays on, day or night, ignoring
    motion and the on period, until an "off" command, a power cut, or the fallback of FR-029.
  - Choosing a brightness or colour MUST make it the on-setting and counts as "on": the light
    fades to it (within about the fade-in time) and the lamp is in manual mode.
  - "Off" MUST fade the light from its current brightness to off at the normal fade-out rate
    and put the lamp back in automatic mode (motion in the dark, FR-018 and FR-023).
  - A fade time sent by Home Assistant MUST be ignored; fades always use the configured
    rates.
- **FR-028**: When the home automation is unreachable, the lamp MUST keep working on motion
  and darkness alone, reconnect automatically when it becomes reachable, and then report its
  current status.
- **FR-029**: If the lamp is in manual mode and Home Assistant has been unreachable for
  10 minutes without interruption (manual-mode fallback time), the lamp MUST return to
  automatic mode. If it is dark (or the time is unknown) and motion was detected within the
  last on period, the light stays on as in automatic mode, with the on period counting from
  the latest detection; otherwise it MUST fade out at the normal fade-out rate.
- **FR-030**: The owner MUST be able to load a new version of the program onto the lamp over
  WiFi; only the first load may require a cable. Updates MUST be protected by a password, and
  an update attempt with a wrong password MUST be refused without interrupting the lamp. After
  an update the lamp MUST restart and behave as after power-up (FR-009, FR-010).
- **FR-031**: The brightness and colour chosen in Home Assistant MUST be kept across power
  cuts, restarts and updates, and MUST be stored only when they change. Whether the light is
  on and the automatic/manual mode MUST NOT be kept (FR-009, FR-030).

### Key Entities

- **Lamp state**: the phase the lamp is in: *off*, *fading in*, *on* (holding full
  brightness), or *fading out*; and its mode: *automatic* (driven by motion and darkness) or
  *manual* (switched on from Home Assistant). It also tracks the current brightness and when
  motion was last detected. After power-up the mode is always automatic.
- **Timing settings**: the fade-in time (default 1 s), on period (default 60 s), fade-out
  time (default 5 s), sensor stabilization period (default 30 s), and manual-mode fallback
  time (default 10 minutes).
- **Strip settings**: the number of LEDs on the addressable strip, and the default on-colour
  and brightness (default: white at maximum, red/green/blue off, brightness 100 %).
- **On-setting**: the brightness and colour (red, green, blue and white levels) last chosen in
  Home Assistant, used for every "on". It starts as the default from the strip settings and is
  kept after a power cut.
- **Location settings**: the latitude and longitude of the installation, used to work out
  sunset and sunrise.
- **Network credentials**: the WiFi network name and password, the encryption key for the
  Home Assistant connection, and the update password, kept separately from the other settings.
- **Status report**: what the lamp tells the home automation: online/offline, light on/off,
  current brightness, motion detected, dark/daylight, and automatic/manual mode.
- **Remote command**: a request from the home automation to switch the light on or off, or to
  choose its brightness and colour.
- **Clock**: the current date and time, and whether it is known (synchronized at least once
  since power-up, from the internet time service or Home Assistant) or unknown.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: In 10 out of 10 trials, the light reaches full brightness within 0.9–1.1 seconds
  after the sensor reports motion.
- **SC-002**: In 10 out of 10 single-detection trials, the light stays at full brightness until
  60–61 seconds after the sensor stops reporting motion, then fades to completely off in
  4.75–5.25 seconds.
- **SC-003**: Across 20 consecutive on/off cycles, an observer standing next to the strip
  reports no visible flicker, steps, or sudden jumps in brightness.
- **SC-004**: During a 30-minute test with movement at least every 45 seconds, the light never
  starts fading out (0 occurrences).
- **SC-005**: In 5 out of 5 trials where motion occurs halfway through a fade-out, the light
  returns to full brightness within 1 second without first going dark.
- **SC-006**: In 5 out of 5 power-cut trials, the lamp is off when power returns, does not
  switch on by itself, and responds normally to motion once the stabilization period has
  passed (in the dark, or at any time while the time is unknown).
- **SC-007**: The lamp keeps the timing of SC-001 and SC-002 after 60 days of continuous
  operation without a restart.
- **SC-008**: The owner can change any timing value and load the updated program over WiFi in
  under 5 minutes, by editing only the settings location and without opening the enclosure.
- **SC-009**: In 10 out of 10 daytime trials with the time known, motion leaves the strip off.
- **SC-010**: On 3 different days, the lamp starts responding to motion within 2 minutes after
  sunset and stops responding within 2 minutes after sunrise, as published for the
  installation location.
- **SC-011**: In 5 out of 5 daytime power-up trials with WiFi switched off, motion turns the
  lamp on; after WiFi is switched back on, the lamp stops responding to daytime motion within
  2 minutes.

- **SC-012**: After first power-up on the home network, Home Assistant shows the lamp as a
  discovered device within 5 minutes; adding it takes only one confirmation and pasting its
  encryption key, with no address or other manual configuration.
- **SC-013**: In 10 out of 10 trials, a status change (light on/off, motion) is shown in the
  home automation within 2 seconds, and an on/off, brightness or colour command from the home
  automation starts the fade within 1 second.
- **SC-014**: With the home automation switched off for 1 hour, the lamp passes SC-001 and
  SC-002, and its status is correct in the home automation within 1 minute after it comes back.
- **SC-015**: In 3 out of 3 trials in daylight, with the light switched on from Home Assistant
  and Home Assistant then stopped, the light starts fading out 10–11 minutes later; in 3 out
  of 3 trials with a 5-minute interruption, the light stays on.
- **SC-016**: In 3 out of 3 trials, a brightness and colour chosen in Home Assistant are shown
  within 2 seconds, used by the next motion-triggered "on", and still used after a power cut;
  a fade-in from off to a chosen brightness below 100 % still takes 0.9–1.1 seconds.

All timing criteria (SC-001 to SC-005) are measured while it is dark.

## Assumptions

- The LED strip is an addressable RGBW strip (SK6812 RGBW type: red, green, blue, and a
  dedicated white LED per pixel; 5 V, one data line) with its own power supply. Because any
  colour can be chosen in Home Assistant, the supply is sized for all four channels at full
  on every LED. Although each LED could be controlled separately, this feature drives them
  all identically (FR-016).
- Brightness and colour are chosen only in Home Assistant; there is no colour control on the
  lamp itself. Without Home Assistant the lamp keeps using the last chosen values.
- The motion sensor is a standard PIR module that reports motion as a simple on/off signal.
  The signal stays on while it senses motion, plus its own hold time. The on period counts
  from the end of that signal, so after a single detection the light stays on for the hold
  time plus 60 seconds. The Grove mini PIR's hold time and retrigger behaviour are not
  published, so they are measured during validation.
- "Minimal 1 minute always on" means the light stays on for 1 minute after the **most recent**
  detected motion ends, so continued movement keeps it on (Story 2).
- Darkness is computed from the time and location, not measured. Local conditions (heavy
  cloud, shade from buildings) do not change the dark period; a light sensor is out of scope.
- A home WiFi network with internet access is within range of the lamp and offers 2.4 GHz
  (the only band the ESP32-C3 supports). WiFi is used to fetch the time from a public
  internet time service and from Home Assistant, and to talk to the home automation.
- The home automation is Home Assistant, running on a Raspberry Pi in the same home network
  as the lamp. All communication with it stays inside the home network; there is no cloud
  service. The lamp connects through Home Assistant's built-in ESPHome integration; no message
  broker or other add-on is needed on the Pi.
- There is no physical override switch and no app of its own; remote control happens only
  through the home automation.
- Timing values are changed by editing the settings and loading the program again over WiFi
  (FR-030); there is no adjustment while the lamp is running.
- The 30-second stabilization period after power-up is a default chosen to cover typical PIR
  warm-up behavior; it can be shortened in the settings if the installed sensor settles
  faster.
- A sensor that is stuck reporting motion keeps the light on; detecting that fault is out of
  scope.
- The target board is an ESP32-C3 running ESPHome firmware, built and loaded from the owner's
  PC.
