# Node 01 — Health Dashboard: Wiring Guide

**Deliverable for Epic 1 Story 1.2.** This is the single source of truth for the pin map and harness. Firmware (`main/board_pins.h` in Story 1.3) references this guide — never the other way around. Any pin change here requires a matching firmware change in the same commit.

## Hardware inventory

| Qty | Part | Role | Notes |
|---|---|---|---|
| 1 | ESP32-S3-WROOM-1 N16R8 DevKitC-1 | MCU | 16 MB flash, 8 MB octal PSRAM, PCB-trace antenna |
| 2 | GC9A01 2.28" round TFT (SPI, 240×240) | Round A (headline), Round B (secondary) | 3.3 V logic, BL pin |
| 1 | ILI9341 2.8" TFT (SPI, 240×320) | Rectangle (chart), **portrait orientation** | 3.3 V logic, BL pin |
| 1 | USB-C 5 V / 2 A wall adapter | Power | Feeds DevKit USB-C |
| — | JST-XH 2-way splitters, silicone ribbon cable, JST-SH connectors | Harness | VCC + GND fan-out |
| *(optional)* | N-channel MOSFET (AO3400 or 2N7002) + 100 Ω + 10 kΩ | Backlight low-side switch | Only if boards have bare-LED BL pins — see §Backlight circuit |

## GPIO allocation — ESP32-S3-WROOM-1 N16R8

**Avoid list (do not use, per ESP32-S3 reference manual + WROOM-1 N16R8 datasheet):**

| GPIO | Reason |
|---|---|
| 0, 3, 45, 46 | Strap / boot mode |
| 19, 20 | USB D+/D− (USB-CDC used for factory flash + serial monitor) |
| 26–32 | Internal SPI flash |
| 33–37 | **Octal PSRAM** — WROOM-1 N16R8 uses OPI PSRAM, these pins are reserved |

**Safe usable GPIOs:** 1, 2, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 21, 38, 39, 40, 41, 42, 43, 44, 47, 48.

## Pin map (locked)

### Shared SPI bus (SPI2 / FSPI host)

| Signal | ESP32-S3 GPIO | Destination | Notes |
|---|---|---|---|
| MOSI | **GPIO11** | All 3 displays (DIN / SDA) | FSPI default |
| SCLK | **GPIO12** | All 3 displays (CLK / SCL) | FSPI default |
| RST  | **GPIO17** | All 3 displays (shared reset) | Active-low, driven high post-boot |

MISO is **not wired** — displays are write-only in this application.

### Per-display signals

| Display | CS (chip-select) | DC (data/command) |
|---|---|---|
| Round A (GC9A01, headline metric) | **GPIO10** | **GPIO14** |
| Round B (GC9A01, secondary metric) | **GPIO9**  | **GPIO21** |
| Rectangle (ILI9341, chart)          | **GPIO8**  | **GPIO7**  |

### Backlight

| Signal | ESP32-S3 GPIO | Destination | Notes |
|---|---|---|---|
| BL_PWM | **GPIO18** | All three BL pins (**direct**, assuming onboard driver) | 8-bit LEDC PWM, 1 kHz. See §Backlight circuit for the rare bare-LED case. |

### Power

| Rail | Source | Destination |
|---|---|---|
| 5 V | USB-C VBUS (DevKit) | Not used by displays |
| 3.3 V | DevKit on-board LDO | All 3 display VCC, via JST-XH 2-way splitters (fan-out) |
| GND | DevKit ground plane | All 3 display GND, via JST-XH 2-way splitters (fan-out) |

**Total signal pins used:** 10 (2 SPI shared + 3 CS + 3 DC + 1 RST + 1 BL). All avoidance rules respected.

## Backlight circuit

### Default path — direct GPIO drive (common case)

Almost all GC9A01 and ILI9341 breakout boards for ESP32 / Arduino have an **onboard transistor** driving the LED array. On those boards, BL is a logic-level input and the GPIO sinks only gate-drive current (< 1 mA per board, ~3 mA total for all three). Well under the ESP32-S3 40 mA per-GPIO limit.

**Wiring:** GPIO18 → fan-out (JST-XH 3-way splitter) → BL pin on each of the three displays.

PWM on GPIO18 modulates brightness; all three displays dim in lockstep (required by FR17/FR18).

### Verify before wiring (30 seconds)

Pick one display board:

- **Multimeter continuity / diode mode** from BL pin to VCC: if you see a resistor-like reading (tens of Ω to a few kΩ) or a diode-like forward drop, there's an onboard driver → **direct GPIO drive is safe**. If it looks like an open pin with no path, the board does bare-LED drive → use the MOSFET variant below.
- **Visual:** a small SOT-23 footprint (marked Q1 or similar) near the BL pin on the board usually means an onboard driver is present.

### Fallback path — low-side MOSFET (only if bare-LED BL)

If the check above shows bare-LED wiring (rare on modern breakouts), each board's BL may draw 20–40 mA at full brightness — 3 × 40 = 120 mA total, exceeding the GPIO limit. In that case:

```
      +3.3 V ──┬── display_A.BL ──┐
              ├── display_B.BL ──┼─── Drain (D)
              └── display_C.BL ──┘
                                  │
                       AO3400 ────┤
                                  │
        GPIO18 ──[ 100 Ω ]── Gate (G)
                              │
                     [10 kΩ] ─┴── GND
                                  │
                        Source (S)── GND
```

- **Q1:** AO3400 (SOT-23, logic-level N-MOSFET) — or 2N7002 for through-hole breadboard.
- **R_gate:** 100 Ω series on gate.
- **R_pulldown:** 10 kΩ from gate to GND (keeps MOSFET off during ESP32 reset / boot).

Single MOSFET gates all three BL → coordinated brightness preserved.

## Wiring diagram

Two views. The **topology view** shows what's shared vs per-display. The **pin-level view** lists every wire as a row.



### Pin-level view

Every wire as a row. Use this as the build checklist.

| # | From (ESP32-S3) | Signal | To (display pin) | Which displays |
|---|---|---|---|---|
| 1 | 3V3 | 3.3 V | VCC | all 3 (via JST fan-out) |
| 2 | GND | GND | GND | all 3 (via JST fan-out) |
| 3 | GPIO11 | MOSI | SDA / SDI | all 3 (via fan-out) |
| 4 | GPIO12 | SCLK | SCL / SCK | all 3 (via fan-out) |
| 5 | GPIO17 | RST  | RST | all 3 (via fan-out) |
| 6 | GPIO18 | BL_PWM | BL / LED | all 3 (via fan-out, direct drive) |
| 7 | GPIO10 | CS (Round A) | CS | Round A only |
| 8 | GPIO14 | DC (Round A) | DC | Round A only |
| 9 | GPIO9 | CS (Round B) | CS | Round B only |
| 10 | GPIO21 | DC (Round B) | DC | Round B only |
| 11 | GPIO8 | CS (Rect) | CS | Rect only |
| 12 | GPIO7 | DC (Rect) | DC | Rect only |

**12 wires total** out of the DevKit (counting 3V3 and GND). 10 are signals; 2 are power.

## Harness assembly notes

1. **Shared-bus trunks.** MOSI, SCLK, and RST each run as a single wire from the DevKit into a **JST-XH 3-way splitter** (or solder-joint fan-out at midpoint), then one leg to each display. Keep all trunk legs the **same length** to avoid reflection on the SPI bus at 40 MHz.
2. **Per-display leaders.** CS and DC are point-to-point — DevKit pin → display pin, one wire each.
3. **Power fan-out.** 3.3 V and GND use separate JST-XH 2-way splitters (never share a connector with signals — less chance of miswire on rebuild).
4. **Ribbon cable.** Silicone 28 AWG ribbon is preferred — flexible inside the faceplate enclosure. Keep length ≤ 150 mm from DevKit to furthest display to stay within SPI trace-length tolerance at 40 MHz.
5. **No metal.** Do not use metal mounting fasteners, foil shielding, or metal ferrules near the DevKit's PCB antenna (preserves WiFi + ESP-NOW reach through plastic enclosure — PRD RF constraint).
6. **Backlight fan-out.** JST-XH 3-way splitter on GPIO18 → one leg to each display's BL pin. (If you're on the rare bare-LED-BL variant, the MOSFET board goes between GPIO18 and the fan-out — see §Backlight circuit.)

## Continuity checklist

Work through before first power-on. Each row = one beep on a multimeter continuity test.

### Shared bus

- [ ] MOSI — GPIO11 → Round A SDA
- [ ] MOSI — GPIO11 → Round B SDA
- [ ] MOSI — GPIO11 → Rect SDI
- [ ] SCLK — GPIO12 → Round A SCL
- [ ] SCLK — GPIO12 → Round B SCL
- [ ] SCLK — GPIO12 → Rect SCK
- [ ] RST — GPIO17 → Round A RST
- [ ] RST — GPIO17 → Round B RST
- [ ] RST — GPIO17 → Rect RST

### Per-display control

- [ ] CS Round A — GPIO10 → Round A CS (and **not** to Round B or Rect CS pins)
- [ ] CS Round B — GPIO9 → Round B CS (and **not** to Round A or Rect CS pins)
- [ ] CS Rect — GPIO8 → Rect CS (and **not** to either round's CS pins)
- [ ] DC Round A — GPIO14 → Round A DC
- [ ] DC Round B — GPIO21 → Round B DC
- [ ] DC Rect — GPIO7 → Rect DC

### Power

- [ ] 3.3 V rail — DevKit 3V3 pin → all three VCC pins
- [ ] GND rail — DevKit GND pin → all three GND pins
- [ ] No continuity between 3.3 V and GND anywhere (shorts check)

### Backlight (direct-drive path)

- [ ] GPIO18 → Round A BL
- [ ] GPIO18 → Round B BL
- [ ] GPIO18 → Rect BL
- [ ] No continuity between GPIO18 and 3.3 V or GND (shorts check)

### Backlight (MOSFET fallback path — only if bare-LED BL)

- [ ] GPIO18 → 100 Ω → MOSFET gate
- [ ] 10 kΩ between MOSFET gate and GND
- [ ] MOSFET source → GND
- [ ] MOSFET drain → all three BL pins (A, B, Rect)
- [ ] No continuity between GPIO18 and any display's BL pin directly

### No miswires

- [ ] No CS line shared between two displays
- [ ] No DC line shared between two displays
- [ ] MOSI and SCLK never cross with CS/DC lines on connectors
- [ ] No signal shorts to 3.3 V or GND

## First power-on protocol

1. Apply USB-C 5 V with **no firmware flashed** (factory DevKit blink demo is fine) and a USB-C ammeter inline.
2. Measure idle current draw: expect < 50 mA (DevKit alone, displays inert since no SPI commands). If > 100 mA with no firmware, suspect a short — power down immediately.
3. Check for thermal anomalies (touch DevKit LDO + MOSFET + each display VCC pin). Nothing should be hot.
4. If clean, flash a bench "hello SPI" utility (see next section).

## Bench verification — "hello SPI" utility

Story 1.2 AC requires each display to respond independently before Story 1.4 wires up LVGL. Minimal test sketch (bench utility only, **not** product firmware):

- For each display: assert CS low, issue the panel's **Read Display ID** command (GC9A01 `0x04 RDDID`, ILI9341 `0x04 RDDIDIF`), read 3 bytes, log over serial. Alternatively, write a solid red fill command sequence; the panel lights up in red.
- Success criterion: all three panels respond independently when their CS is asserted, and **none** respond when someone else's CS is asserted.

A reference bench utility belongs under `nodes/node-01-health-dashboard/firmware/bench/hello_spi/` — to be committed as part of Story 1.2 closeout or early Story 1.4 scaffolding.

## Reference photo

*Placeholder — committed post-assembly.* Add a labelled photo of the completed harness at `nodes/node-01-health-dashboard/hardware/wiring/harness-photo.jpg` with callouts matching this guide's GPIO numbers.

## Change log

| Date | Change | By |
|---|---|---|
| 2026-04-23 | Initial pin map + harness diagram + continuity checklist | Kamal (via Claude) |
| 2026-04-23 | Switched backlight to direct GPIO drive (onboard-driver common case); MOSFET moved to optional fallback | Kamal (via Claude) |
