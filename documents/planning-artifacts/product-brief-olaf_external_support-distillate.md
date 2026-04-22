---
title: "Product Brief Distillate: OLAF External Support"
type: llm-distillate
source: "product-brief-olaf_external_support.md"
created: "2026-04-18"
purpose: "Token-efficient context for downstream PRD / firmware / hardware spec work"
---

# OLAF External Support — Detail Pack

## Project Identity & Scope Signals

- **Project name:** OLAF External Support (working directory: `olaf_external_support`)
- **Type:** Personal, non-commercial hardware project. Kamal + family. Oberrohrdorf, Switzerland.
- **Platform thesis:** Family of custom ESP32-based hardware nodes OLAF commands via MQTT — "OLAF's hands and eyes in the rooms he can't reach."
- **Currently in scope:** Health Dashboard (Node 1). Hardware, firmware, MQTT framework. 3D-printed enclosure already exists.
- **Currently out of scope (deferred, not abandoned):** Other node archetypes (sensor, controller, media); voice / conversational interaction; multi-room replication; data ingestion pipeline design; compositional firmware primitives; open-source publishing.
- **Timeline:** No fixed deadline for the project overall. Hardware build targeted within ~2 days.

## OLAF (the brain) — External Context

- **Repo:** https://github.com/kamalkantsingh10/OLAF
- **What it is:** Open-source companion robot — personality-first embodied AI, self-balancing two-wheel base, SLAM navigation.
- **Stack:** Raspberry Pi 5 16GB + Hailo AI Kit (26 TOPS), Ubuntu 24.04, ROS2 Jazzy, Sunfounder Fusion HAT+ for hardware control. Local Whisper STT (<200ms) + cloud Claude/GPT-4. SQLite conversation history.
- **OLAF's own body:** 2 ESP32 controllers (eyes @60 FPS, balance @200 Hz), 3-DOF neck, expressive eyes, articulated ears, heart display, 24 RGB LEDs.
- **Status:** Still under active development. External Support nodes must be useful before OLAF is fully mature.
- **Why External Support exists vs. extending OLAF's body:** OLAF's displays are too small for 30-day charts; OLAF can only be in one room; voice is expensive for passive glances; family members shouldn't need to ask OLAF each time.

## Platform Design Principles (all load-bearing)

- **AI-directed hardware:** Firmware ships with a vocabulary of rendering/control primitives. OLAF composes them at runtime via MQTT. Switching chart types or triggering pre-defined animations is an MQTT message, not a firmware release. Genuinely new primitives (e.g., a novel animation) require OTA — but one OTA unlocks that primitive for all future OLAF directives.
- **Dumb renderer, smart brain:** Zero content-domain logic in firmware. Firmware has no knowledge of what "glucose" or "fasting" means. OLAF pushes content, status, thresholds; node obeys. *Caveat: firmware still has significant rendering-domain logic — this is not "no logic," it is "no domain semantics."*
- **One spine, many forms:** Every node uses ESP32 + MQTT (data) + ESP-NOW (wake) + last-known-good caching + PWM brightness + deep sleep. New node types inherit the spine; only form factor and primitive vocabulary change.
- **Teachable longevity:** A 2-year-old node should still be directable by OLAF in ways Kamal didn't anticipate — within the primitive vocabulary shipped in firmware.
- **Ambient, not conversational:** Principled rejection of "ask the AI every time." Dashboard is pushed-to, not asked. No voice interaction with nodes in current scope.
- **Family-usable:** Nodes must work for non-builders. Glance-readable, ritual-free, no phone required.
- **Framework for OLAF leeway:** MQTT schema designed so OLAF can push arbitrary valid content without firmware changes — category definitions, metrics, thresholds, chart types, rotation timing, brightness, pause state all pushed.

## Health Dashboard — Technical Spec (folded from v1.1)

### Concept

- One category shown across all three displays simultaneously. Round A = headline metric, Round B = secondary metric, Rectangle = chart for that category.
- All three screens transition together to next category on rotation interval (default 10s). Viewer always sees coherent single-domain snapshot.
- Category name/index shown briefly as overlay at start of each slot (configurable).
- Transition style configurable: instant / wipe / fade.

### Reference Categories (pushed by OLAF, not hardcoded)

| ID | Category | Round A | Round B | Chart |
|---|---|---|---|---|
| 1 | Body Weight | Current weight (e.g. 84.2 kg) | 7-day trend (e.g. -0.4 kg this week) | 30-day line graph with moving average |
| 2 | Glucose | Last reading (e.g. 5.4 mmol/L) | Daily average (e.g. 5.8 mmol/L) | 14-day scatter plot with range bands |
| 3 | Cycling | This week (e.g. 47.3 km) | Last ride (e.g. 22.1 km · 2 days ago) | 8-week bar chart (km per week) |
| 4 | Fasting | Last window (e.g. 16:30 hh:mm) | Weekly average (e.g. 15.8 hrs/day) | 14-day fasting window bar chart |

### Hardware BOM

- **MCU:** ESP32-S3-WROOM-1 N16R8 DevKit, PCB trace antenna (16 MB flash, 8 MB OctalPSRAM) — *upgraded from ESP32 WROOM in architecture 2026-04-22*
- **Round Display ×2:** 2.28" round TFT, GC9A01 driver, 240×240px, SPI
- **Rectangle Display ×1:** 2.8" TFT, ILI9341 driver, 320×240px, SPI
- **Power:** USB-C 5V/2A wall adapter, always-on, no battery
- **Radios:** 2.4GHz WiFi (data) + ESP-NOW (wake/sleep), shared antenna
- **Enclosure:** 3D printed (already exists). Plastic/acrylic only — no metal (preserves WiFi/ESP-NOW)

### SPI Bus Wiring

- **Shared across all 3:** MOSI (1 GPIO), SCLK (1 GPIO), GND/VCC (fan-out via JST splitter), BL (backlight, PWM via `ledcWrite()` 0–255), RST (shared or per-display)
- **Per display:** CS (3 GPIO), DC (3 GPIO)

### Ring Rendering (GC9A01, 240×240 round)

- Coloured annulus at display edge: outer radius 118px, inner 106px, centre (120, 120). 12px band inside the physical bezel.
- Metric value, unit, label, timestamp rendered in centre — unaffected by ring.
- Ring redraws independently on status change; centre content does not refresh.

### Status Colour System

| Status | Colour | Hex | Meaning |
|---|---|---|---|
| Good | Green | #00C853 | Value within healthy target range |
| Warning | Amber | #FFB300 | Value approaching threshold — monitor |
| Bad | Red | #D32F2F | Value outside healthy range — attention needed |
| Unknown | Grey | #616161 | No data yet, or data too stale |

### Threshold Reference Values (pushed by OLAF, not hardcoded in firmware)

| Metric | Good | Warning | Bad |
|---|---|---|---|
| Body Weight | ±1 kg of target | +1 to +4 kg over | >+4 kg over |
| Glucose | 4.0–7.0 mmol/L | 7.0–10.0 mmol/L | <3.9 or >10.0 mmol/L |
| Cycling (weekly) | >60 km | 30–60 km | <30 km |
| Fasting window | >14 hrs | 12–14 hrs | <12 hrs |

Glucose unit is configurable (mmol/L vs mg/dL) via MQTT payload field. Target weight is a personal parameter. Clinical ranges are defaults but overridable.

### Power Model

| State | Draw | Trigger |
|---|---|---|
| Active — full brightness | ~380 mA | OLAF active |
| Active — 50% brightness | ~200 mA | Dimmed |
| Backlight 0% (visually off) | ~100 mA | `health/brightness: 0` — ESP32 still WiFi-connected |
| Deep sleep | ~0.5 mA | `health/cmd: shutdown` |

Backlight 0% is the primary idle state. Deep sleep is for overnight / OLAF-off only.

## Communication Architecture

### Dual-Channel Design

| Channel | Direction | Protocol | Router needed? |
|---|---|---|---|
| Wake | OLAF → ESP32 (from deep sleep) | ESP-NOW broadcast | No — radio layer only |
| Sleep | OLAF → ESP32 (while awake) | WiFi + MQTT | Yes |
| Data | OLAF → ESP32 (while awake) | WiFi + MQTT | Yes |

**Strict channel separation:** ESP-NOW = wake only. WiFi = everything else including sleep. Once ESP32 enters deep sleep, WiFi is fully off — only the RTC ESP-NOW receiver listens. A sleep command sent after ESP32 is already asleep is silently lost.

### State Machine

- **Deep sleep → Awake:** OLAF ESP-NOW wake packet → RTC wakes CPU → WiFi connects → MQTT subscribes → re-renders last-known state (OLAF re-pushes all current values on startup).
- **Awake → Deep sleep:** OLAF MQTT `health/cmd: "sleep"` → backlight off → stop rendering → MQTT disconnect → WiFi off → deep sleep (~10µA ideal, ~0.5mA measured).

### MQTT-Styled Payload Contract (logical; v2 wire is ESP-NOW opcodes via Master ESP32)

> *The topic names and payload shapes below describe the logical contract OLAF produces and the target renders. In architecture v2 (2026-04-22), the physical wire is ESP-NOW opcode envelopes between Master ESP32 and target, routed through a Python MCP server — not MQTT. Topic names below map 1:1 to opcodes; schema is unchanged.*

### Topic Structure

| Topic | Payload | Description |
|---|---|---|
| `health/cmd` | `"sleep"` \| `"shutdown"` | Sleep command over WiFi (while awake) |
| `health/brightness` | 0–255 | Backlight PWM |
| `health/rotation_interval` | Integer (seconds) | Time per category, default 10 |
| `health/category_order` | `[1,2,3,4]` array | Rotation sequence |
| `health/active_categories` | Subset array | Enable/disable categories |
| `health/pause_category` | `0` (resume) \| `1-4` (freeze) | Hold display on one category |
| `health/category/{n}/metric_a` | `{value, unit, label, ts, status, thresholds}` | Round A content |
| `health/category/{n}/metric_b` | `{value, unit, label, ts, status, thresholds}` | Round B content |
| `health/category/{n}/chart` | `{type, points[], unit, label, x_label}` | Chart data |

### Example Payloads

```json
// health/category/1/metric_a
{"value":"84.2","unit":"kg","label":"Body Weight","ts":"this morning","status":"good","thresholds":{"good":[0,86],"warn":[86,90],"bad":[90,999]}}
```

```
// health/rotation_interval
10
```

### Last-Known-Good Persistence

- ESP32 caches last received payload for every topic in RAM.
- On deep-sleep wake, OLAF re-pushes all current values as part of startup sequence.
- Between updates, display continues showing last received data.
- `ts` field in metric payload communicates data age to viewer.

### Multi-node (future, not in current scope)

- Multiple nodes on same broker subscribe to `health/#` — all stay in sync automatically.
- Per-node overrides: `health/{node_id}/#` subtopics.

## Firmware Architecture (module breakdown)

| Module | Responsibility |
|---|---|
| ESP-NOW receiver | Listens for wake/sleep packets during deep sleep via RTC |
| WiFi / MQTT client | Connects on wake, subscribes `health/#`, maintains connection |
| Category manager | Holds 4-category dataset in RAM, tracks active category index |
| Rotation timer | Advances category index every N seconds (N from `health/rotation_interval`) |
| Round renderer | Draws status ring + large number/unit/label/timestamp on GC9A01 |
| Chart renderer | Line or bar chart on ILI9341 from `points[]` array |
| Transition controller | Coordinates simultaneous clear+redraw across all 3 displays on category change |
| Backlight controller | PWM via `ledcWrite()`, responds to `health/brightness` |
| Sleep manager | On `health/cmd: "sleep"`: backlight 0 → stop rotation → MQTT disconnect → WiFi off → deep sleep. ESP-NOW RTC receiver stays active. |

**Firmware invariant:** ESP32 has no knowledge of what health categories *mean*. It stores N category slots, each with metric_a / metric_b / chart payloads. OLAF defines all content. Adding/renaming/reordering categories = MQTT publish only, zero firmware change.

## Offline & Degraded Behaviour Contract

- **OLAF unreachable, broker up:** Keep rendering last-known-good. Dim status ring after N minutes without update to signal staleness.
- **Broker down, WiFi up:** Same — last-known-good is the only truth.
- **WiFi down:** Same. ESP-NOW continues listening for OLAF wake/reset.
- **Fresh boot, no prior data:** Grey status rings, "waiting for OLAF" placeholder — never blank or misleading numbers.
- **Rule:** A node must never be more wrong than honestly stale.

## Rejected Ideas / Out of Scope (do not re-propose)

- **Voice / conversational interaction with dashboard** — OLAF does not speak to, or receive voice commands about, the dashboard in this project. No "OLAF, show my glucose" voice-freeze flow.
- **ESPHome as firmware substrate** — Considered. Rejected for Health Dashboard because custom three-screen rotating layout + status rings + coordinated transitions would require custom C++ components anyway. Revisit if a sensor/controller node is ever built.
- **Home Assistant as primary integration** — Considered as substrate (device discovery, long-term storage). Out of scope for this project; OLAF is the sole controller.
- **Matter / Thread** — Deferred; not relevant for a display node.
- **Compositional firmware primitives** (full drawing DSL — draw_arc, fill_to, tween, easing) — Deferred. Semantic vocabulary for v1 (chart_type from fixed catalog). MQTT schema designed to remain forward-compatible with compositional additions later.
- **Multi-room dashboard replication** — Architecturally supported via `health/{node_id}/#` but not built.
- **Other archetypes (presence sensor, IR remote, MP3 player)** — Architecturally planned in the platform shape but not committed in this project.
- **Choreographed cross-node experiences** (sunrise routines, movie-mode cascades) — Only meaningful when multiple nodes exist.
- **Open-source documentation / public packaging** — Not a design driver.
- **Commercial product packaging** — Not a goal.
- **Wearables / pocket nodes** — Not in scope.
- **Battery power** — Rejected; USB-C always-on is the model.

## Open Questions / Decisions Deferred to Firmware Phase

- **Firmware vocabulary shape:** Semantic (current leaning — fixed catalog of chart types + animations) vs Compositional (OLAF sequences drawing primitives) vs Hybrid. *Decision: start semantic; design MQTT schema so compositional primitives can be added later without breaking change.*
- **Data ingestion into OLAF:** Deliberately deferred. Dashboard's MQTT framework is neutral on data origin. OLAF can use public libraries for wearable integration, voice entry, CSV, webhooks — whatever he chooses. Dashboard does not care.
- **Privacy surface:** Wall-mounted dashboard visible to guests/visitors/cleaners. Options to explore: privacy mode toggle; ambiguous display (status ring only, no number); OLAF-controlled discretion directives based on presence.
- **OTA firmware channel:** Assumed. Design explicitly before vocabulary expansion becomes expensive.
- **Inter-node coordination:** Not urgent in current scope. If multiple nodes ever exist, coordinated behaviours may need richer orchestration than topic subscription.
- **Category overlay timing:** How long to show category name/index at start of each slot. Configurable — default TBD.
- **Transition animation:** Instant / wipe / fade. Default is instant per v1.1; configurable via OLAF.
- **Chart colour theming per category:** Pushed by OLAF in chart payload.

## Known Next-Phase Work Items

- **Hardware build (days):** ESP32 + 3× SPI display wiring inside existing 3D-printed enclosure. USB-C power. SPI fan-out.
- **Visual / on-screen design:** Typography, layout, colour palette, status ring styling, transition animations, chart aesthetics. Calm enough to live on a wall, clear enough to be glanced at.
- **Firmware (own pace):** Implement primitive vocabulary, MQTT framework, offline/degraded behaviours, OTA channel, ESP-NOW wake handler, sleep manager.
- **OLAF-side publisher:** In parallel, OLAF gains the ability to push all health topics. Develops as OLAF matures.

## Requirements Hints (for PRD consumption)

- Firmware must support rotation, transitions, brightness control, sleep/wake, last-known-good cache, ESP-NOW wake, MQTT pub/sub — all via topic-driven configuration.
- Firmware must never evaluate health thresholds — OLAF resolves status server-side and pushes the status string.
- Firmware must render status as ring (GC9A01) independently of centre metric content.
- Firmware must handle missing OLAF gracefully with staleness indication, never blank-on-absent-data.
- MQTT schema must be extensible: adding category 5 requires no firmware change; changing a chart type requires no firmware change; adding a new status colour does require firmware (acceptable).
- Hardware must preserve RF performance (no metal enclosure, shared antenna for WiFi+ESP-NOW).
- Power: always-on USB-C; backlight-0 is the primary "idle"; deep sleep is exceptional.

## User / Audience Context

- **Kamal:** Builder, primary user, owner of OLAF project. Intermediate skill level (per BMM config). Works in evenings/weekends as personal project. Based in Oberrohrdorf, Switzerland.
- **Family:** Secondary users. Will use once OLAF matures. "Family-usable" means glanceable, zero-explanation, no phone.
- **Not an audience:** Commercial customers, open-source community, other builders. Documentation may go public someday but is not a design driver.
