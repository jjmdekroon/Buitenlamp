# Contract: Hardware, Pin Map, and Wiring

**Feature**: [../spec.md](../spec.md) | **Covers**: constitution Principle I (fail-safe hardware)
and Principle V (pin map), FR-009, FR-012 | **Background**: [../research.md](../research.md)
R2, R4, R5

## Pin map (must match the substitutions in `esphome/buitenlamp.yaml`)

| ESP32-C3 pin | XIAO ESP32C3 label | Connects to | Direction | Electrical notes |
|--------------|--------------------|-------------|-----------|------------------|
| GPIO4 (`pin_pir`) | D2 | Grove mini PIR **SIG** | input, internal pull-down (`mode: {input: true, pulldown: true}`) | 3.3 V logic; the pull-down makes a missing sensor read "no motion" |
| GPIO5 (`pin_led_data`) | D3 | 74AHCT125 **1A** | output (RMT, `esp32_rmt_led_strip`) | 10 kΩ pull-down to GND, so the line stays low during boot |
| 3V3 | 3V3 | Grove mini PIR **VCC** | power out | **never connect the PIR to 5 V**, because SIG would then exceed 3.3 V |
| 5V / VIN | 5V | 5 V supply + | power in | see "USB and external 5 V" below |
| GND | GND | common ground: supply −, strip GND, PIR GND, 74AHCT125 GND | — | one star point near the strip feed |

## Level shifter (74AHCT125, 5 V powered)

| 74AHCT125 pin | Connects to |
|---------------|-------------|
| VCC (14) | 5 V |
| GND (7) | GND |
| 1OE (1) | GND (always enabled) |
| 1A (2) | GPIO5 |
| 1Y (3) | 330 Ω resistor → strip **DIN** |
| 2A, 3A, 4A (5, 9, 12) | GND, so unused inputs don't float |
| 2OE, 3OE, 4OE (4, 10, 13) | 5 V, so unused outputs are disabled |

## LED strip (SK6812 RGBW, 5 V)

| Strip wire | Connects to |
|------------|-------------|
| 5V | 5 V supply +, through a fuse; 1000 µF / ≥ 10 V capacitor between 5V and GND at the feed point |
| DIN | 330 Ω ← 74AHCT125 1Y |
| GND | common GND |

**Power supply size** (updated 2026-11-02): `num_leds × 0.080 A × 1.2 + 0.3 A`. Any colour
can now be chosen in Home Assistant, so all four channels (R, G, B and W, about 20 mA each)
can be at full on every LED at once. For example, 26 LEDs need 2.8 A and 60 LEDs 6.1 A. Use a
fuse rated at about 1.25 × the calculated current. Strips longer than about 2 m should get
5 V fed in at both ends.

The earlier white-only sizing (`num_leds × 0.020 A × 1.2 + 0.3 A`) is only safe if you never
choose a colour with more than the white channel at full.

The 20 mA per channel is an estimate. Measure the real current with all four channels at full
during the hardware gate and record it in the quickstart results.

## USB and external 5 V

When the board is connected to a PC over USB for programming **and** the external 5 V supply
is also connected, the two supplies can fight each other. Either disconnect the external 5 V
wire from the board's 5V pin while USB is plugged in, or check that the board has a protection
diode between USB 5 V and the 5V pin.

## Outdoor installation

- Use an IP65 or better enclosure for the electronics, with cable glands.
- The PIR must look out through an open window or a PIR-transparent lens. **PIR sensors cannot
  detect through glass or clear plastic.**
- The Grove mini PIR reaches **about 2 m** (research R5). Aim it at the spot where people pass
  closest to it.
- Keep the data wire from the 74AHCT125 to the strip short, ideally under 1 m. Put the
  ESP32-C3 near the start of the strip.

## Bill of materials

| Qty | Part |
|-----|------|
| 1 | ESP32-C3 board (DevKitM-1, Seeed XIAO ESP32C3, or similar) |
| 1 | Seeed Studio Grove mini PIR motion sensor (101020353) |
| 1 | SK6812 RGBW 5 V LED strip, `num_leds` pixels |
| 1 | 74AHCT125 (DIP-14) or 74AHCT1G125 |
| 1 | 5 V power supply, sized as above |
| 1 | fuse and holder, sized as above |
| 1 | 1000 µF / ≥ 10 V electrolytic capacitor |
| 1 | 330 Ω resistor |
| 1 | 10 kΩ resistor |
| — | IP65 enclosure, cable glands, wire |
