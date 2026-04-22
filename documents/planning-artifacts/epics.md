---
stepsCompleted: [1, 2, 3, 4]
status: complete
completedAt: 2026-04-23
inputDocuments:
  - documents/planning-artifacts/prd.md
  - documents/planning-artifacts/architecture.md
  - documents/planning-artifacts/ux-design-specification.md
  - documents/planning-artifacts/prd-validation-report.md
project_name: olaf_external_support
author: Kamal
date: 2026-04-22
milestoneStructure: [M1, M2, M3, M4]
epicCount: 5
storyCount: 40
frCoverage: 41/41
---

# olaf_external_support — Epic Breakdown

## Overview

This document provides the complete epic and story breakdown for **olaf_external_support**, decomposing requirements from the PRD, UX Design Specification, and Architecture Decision Document (v2) into implementable stories, organised around four user-directed milestones that grow complexity step by step.

**Milestones (user-directed, 2026-04-22):**

1. **M1 — Target bring-up.** One target node (Node 01 / Dashboard A) renders static/dummy values on all three displays; LVGL + `olaf_dark` theme + tokens operational; OTA pipeline stood up end-to-end over USB/WiFi. No master, no agents. *Simple.*
2. **M2 — Master + first link.** Master ESP32 (Node 00) comes online; establishes first ESP-NOW peer link with target; push a single BRIGHTNESS or REFRESH_SLOT op and see the target react. *First proof the two boxes talk.*
3. **M3 — Full Master↔Target protocol.** Complete ESP-NOW envelope + opcode set (WAKE, SLEEP, REFRESH_SLOT, ROTATION_CONFIG, BRIGHTNESS, WIFI_UP/DOWN, OTA_BEGIN, DIAG_REQUEST, events). Deep sleep + RTC wake + on-demand WiFi TCP + degraded-state rendering. Target is fully dumb-renderable over the wire. *Protocol complete.*
4. **M4 — MCP server + agent directability.** Python MCP server with WebSocket client to master, 6 tools for `dashboard_a`, resources for fleet status, logical-name→MAC routing. OLAF / Claude / scripts can direct the fleet. *Agent-facing.*

Sequence logic: each milestone can be demoed standalone. Later milestones do not refactor earlier ones — they bolt on.

## Requirements Inventory

### Functional Requirements

Extracted from `prd.md` (41 FRs, 8 categories). Where FR language still references "MQTT topics", that maps to ESP-NOW envelope opcodes per architecture v2 — logical contract unchanged.

**Content Rendering**
- **FR1:** Render a **headline metric** (value, unit, label, timestamp) on the upper round display for the currently active category.
- **FR2:** Render a **secondary metric** (value, unit, label, timestamp) on the lower round display for the currently active category.
- **FR3:** Render a **chart** (type, data points, axis label, unit) on the rectangular display for the currently active category.
- **FR4:** Render a **status ring** around each round display using a status token from OLAF (good / warning / bad / unknown).
- **FR5:** Render the **timestamp / staleness indicator** alongside a metric, using human-readable text supplied in the metric payload.
- **FR6:** Centre content and status ring render **independently** — ring status change does not trigger centre redraw and vice versa.
- **FR7:** Render charts for each **supported chart type in the v1 semantic catalog** (line, scatter, bar).
- **FR8:** Render using the **typography and colour palette defined in the UX Design Specification**.

**Category Management & Rotation**
- **FR9:** Store **N category slots** in memory, each holding the most recent `metric_a`, `metric_b`, and `chart` payloads.
- **FR10:** Accept a category payload for **any category slot** (current or future) without firmware modification; firmware treats all slots uniformly.
- **FR11:** **Rotate through active categories** on a cadence defined by a rotation interval directive.
- **FR12:** **Re-order the rotation sequence** in response to a category-order directive.
- **FR13:** **Enable or disable individual categories** in the rotation in response to an active-categories directive.
- **FR14:** **Pause rotation** on a specific category in response to a pause directive and resume when released.
- **FR15:** **Transition all three displays simultaneously** when the active category changes (no per-display stagger).
- **FR16:** Support **instant transitions** as the default and only MVP transition style.

**Dashboard Configuration**
- **FR17:** **Adjust backlight brightness** across all three displays in response to a brightness directive (0–255 range).
- **FR18:** Treat **backlight = 0** as the primary idle state — displays off, node remains reachable and responsive.
- **FR19:** Apply **configuration directives immediately** on receipt without requiring restart.

**State & Power Management**
- **FR20:** **Enter deep sleep** in response to a sleep command — backlight off, rendering stops, WiFi powers down; only ESP-NOW RTC receiver remains active.
- **FR21:** **Wake from deep sleep** in response to an ESP-NOW wake packet, reconnect, re-render last-known-good from master re-push.
- **FR22:** Only sleep/wake on **explicit master directives** — no internal schedule or timer triggers sleep.
- **FR23:** **Reject ESP-NOW wake packets** from any MAC other than the whitelisted peer (master's MAC).

**Network & Communication**
- **FR24:** **Connect to 2.4 GHz WiFi** using credentials provisioned prior to operation (factory NVS). *Target: on-demand only (v2).*
- **FR25:** **Receive opcode envelopes** over ESP-NOW from master (v2 supersedes MQTT broker subscribe).
- **FR26:** **Re-associate with WiFi / re-establish ESP-NOW peering** automatically after disconnection without operator intervention.
- **FR27:** **Receive and parse** every opcode defined in the ESP-NOW wire protocol (commands, configuration, per-category metric_a / metric_b / chart).
- **FR28:** **Ignore unknown fields** in payloads rather than rejecting or crashing. Unknown status values render as "unknown" (grey).
- **FR29:** **Receive ESP-NOW wake packets** while in deep sleep without requiring WiFi.

**Degraded & Recovery Behaviour**
- **FR30:** **Cache the last-received payload** for every topic/slot in RAM (last-known-good cache).
- **FR31:** **Continue rendering from LKG** when master becomes unreachable or ESP-NOW link drops.
- **FR32:** **Indicate staleness** by dimming status rings after a configurable period without fresh data for a given category.
- **FR33:** Render a **distinct "no data" placeholder** on fresh boot — grey rings, dash/blank, "waiting for OLAF" indication. Never fabricate values.
- **FR34:** **Never display numeric values without their corresponding timestamp and status indication** — the three render as a single unit.

**Firmware Lifecycle**
- **FR35:** Operator can **provision the node** with WiFi credentials and master MAC via factory flash (NVS pre-write).
- **FR36:** Operator can **trigger an OTA firmware update** by directing via master (binary URL / chunk stream + expected SHA-256).
- **FR37:** **Download, verify, and flash** a new firmware image; activate on next boot; roll back on failed boot.
- **FR38:** Render a **visually distinct "updating" state** during download and flash — never blank.
- **FR39:** **Boot cleanly from first power-on** without operator interaction beyond one-time provisioning.

**Observability**
- **FR40:** Publish a **Last Will / online-offline status** (`status: offline` on link loss) observable by master / MCP.
- **FR41:** Publish **firmware version** on boot for OTA validation.

### NonFunctional Requirements

Extracted from `prd.md`.

**Performance**
- **NFR-P1:** Glance-to-read primary metric + status ≤ **2 seconds** from 3 m.
- **NFR-P2:** Wake-to-first-render ≤ **3 seconds**.
- **NFR-P3:** Transition coordination — all three displays begin redraw within ≤ **50 ms** of each other.
- **NFR-P4:** Config directives take effect ≤ **500 ms** after receipt.
- **NFR-P5:** Single-display redraw (chart + ring + text) ≤ **100 ms**.
- **NFR-P6:** Boot-to-first-render on fresh power-on ≤ **5 seconds**.

**Reliability**
- **NFR-R1:** Uptime ≥ **99%** over rolling 30 days.
- **NFR-R2:** Automatic recovery ≤ **60 s** after network restoration.
- **NFR-R3:** Zero tolerance — blank displays, fabricated values, or misleading "current" indicators under any degraded condition.
- **NFR-R4:** No memory leaks degrading rendering over 7-day continuous run.
- **NFR-R5:** Failed OTA never bricks the node — rollback mandatory.
- **NFR-R6:** Malformed/truncated payloads never crash firmware; logged and discarded; LKG retained.

**Power & Thermal**
- **NFR-Pw1:** Active full brightness ≤ **420 mA** @ 5 V.
- **NFR-Pw2:** Active 50% brightness ≤ **220 mA** @ 5 V.
- **NFR-Pw3:** Backlight-0 idle ~ **100 mA ±10%** @ 5 V.
- **NFR-Pw4:** Deep sleep ≤ **0.5 mA** @ 5 V.
- **NFR-Pw5:** Enclosure surface ≤ **20 °C above ambient**; absolute ≤ **60 °C**.
- **NFR-Pw6:** Cold-start inrush within USB-C 2 A envelope.

**Security**
- **NFR-S1:** Credentials in NVS, never hardcoded in redistributable binaries.
- **NFR-S2:** Plain TCP on LAN acceptable; TLS supported if broker ever external.
- **NFR-S3:** OTA SHA-256 verification mandatory; mismatch aborts flash.
- **NFR-S4:** ESP-NOW MAC whitelist gates wake packets.
- **NFR-S5:** No unauthenticated remote surfaces in production build.
- **NFR-S6:** Secrets redacted or absent from all log output.

**Integration**
- **NFR-I1:** Parse every opcode/topic in the platform schema without firmware change for additions reusing existing field types.
- **NFR-I2:** Forward schema compatibility — new category slots / chart types / optional fields must be renderable by already-deployed firmware.
- **NFR-I3:** Unknown fields ignored, missing required fields default to unknown/grey, unknown enum values render safely.
- **NFR-I4:** Target parses ESP-NOW envelopes per `docs/espnow-wire-protocol.md`; master exposes WebSocket JSON per `docs/master-ws-protocol.md`.
- **NFR-I5:** Wake packet format frozen for MVP; changes require coordinated master + target release.
- **NFR-I6:** Node functions identically whether driven by real OLAF or stand-in publisher emitting compliant payloads.

**Maintainability**
- **NFR-M1:** Kamal-in-6-months — adding a new chart type is a single-file change in the chart module.
- **NFR-M2:** Module boundaries match the architecture breakdown 1:1, enforced by file organisation.
- **NFR-M3:** Zero references to category-specific strings ("glucose", "weight", "cycling", "fasting") in firmware source outside test fixtures — verified by grep at every commit.
- **NFR-M4:** Platform-shared modules (ESP-NOW, WiFi, LKG cache, sleep manager, OTA) isolated from Node-1-specific modules — Node 2 must copy-paste without refactor.

**Aesthetic & UX Quality**
- **NFR-U1:** Rendering matches UX spec typography scales, weights, exact hex palette.
- **NFR-U2:** Status ring geometry matches UX spec (outer r=118, inner r=106, centre 120,120, 12 px band).
- **NFR-U3:** Transition and state-change animations use UX spec durations and easing.
- **NFR-U4:** OLED-deep-black backgrounds preserve the three-displays-as-one-faceplate effect.

### Additional Requirements

Extracted from `architecture.md` (v1 + v2). These shape implementation and epic sequencing.

- **AR1 (Scaffold):** PlatformIO + ESP-IDF 5.5 + LVGL 9.5 scaffold is the committed starter. Initialization is the first implementation story.
- **AR2 (Partitions):** Dual-partition OTA (`app0` / `app1`) + coredump + SPIFFS layout committed to `partitions.csv`.
- **AR3 (PSRAM framebuffer):** Full-screen double-buffer in PSRAM per display (~1.8 MB total); SPI DMA ping-pong (~4 KB) in internal SRAM.
- **AR4 (LVGL integration):** `esp_lvgl_port` multi-display registration; single LVGL render task pinned to core 1 serving all three `lv_disp_t`; WiFi/network/app on core 0.
- **AR5 (Panel drivers):** `esp_lcd_gc9a01` and `esp_lcd_ili9341` via shared SPI bus with per-display CS/DC; ILI9341 operated in portrait (90° rotation).
- **AR6 (NVS schema):** Factory-written keys `wifi/ssid`, `wifi/pwd`, `mqtt/host`, `mqtt/port`, `olaf/mac`, plus per-target ESP-NOW key. Read-only in field.
- **AR7 (Factory flash):** `make factory-flash` one-shot serial channel. No captive portal, no runtime re-provisioning. Serial console read-only in production build.
- **AR8 (Repo shape):** Monorepo with `spine/` (shared components) + `nodes/node-00-master/` + `nodes/node-01-health-dashboard/` + `mcp/` + `docs/`.
- **AR9 (Spine components):** `net_espnow`, `net_espnow_frag`, `net_wifi_lifecycle`, `net_wifi_persistent`, `net_ws_server`, `net_tcp_client`, `net_tcp_server`, `store_lkg`, `ota_manager`, `sleep_manager`, `diag_manager`, `util_backoff`, `util_json`.
- **AR10 (Master components):** `ws_mcp_endpoint`, `routing_table` (logical_name ↔ MAC ↔ state), `transport_selector` (ESP-NOW vs WiFi), `offline_queue` (buffer for sleeping targets), `ota_streamer`.
- **AR11 (Target components):** `target_dispatcher`, `ui_tokens`, `ui_theme`, `ui_composition_round`, `ui_composition_rect`, `render_round`, `render_chart`, `render_transition`, `render_lifecycle`, `category_store`, `category_rotator`, `backlight_controller`.
- **AR12 (Master hardware):** ESP32-C3 Mini, always-on USB-powered, WiFi persistent + ESP-NOW master role.
- **AR13 (Master transport selector):** ESP-NOW for wake/sleep/config/typical data (≤2 KB fragmented); WiFi TCP on-demand for OTA (~1–2 MB) and sustained log streams.
- **AR14 (ESP-NOW envelope):** 2 B magic + 1 B opcode + 2 B seq/fr + 2 B body_len + 0–240 B JSON body. Opcodes 0x01–0x09 + 0x80+ events. Fragmentation via `seq/fr` field.
- **AR15 (ESP-NOW encryption):** AES-CCM via ESP-NOW built-in peer encryption; per-target key provisioned at factory flash.
- **AR16 (WiFi on-demand):** Target enables WiFi only on master `WIFI_UP`; replies with `{ip, port}` via ESP-NOW; master opens TCP; auto-teardown on idle timeout or explicit `WIFI_DOWN`.
- **AR17 (MCP ↔ Master WebSocket):** WS server on master port 8080; JSON `{id, op, target, action, payload}` command frames + `{id, ok, transport, latency_ms}` responses + `{event, …}` events. No auth for MVP.
- **AR18 (MCP server):** Python (HTTP/SSE) in `mcp/olaf_mcp/`. WebSocket client to master (`master_client.py`). Auto-discovery tools registrar. `systemd` service.
- **AR19 (MCP tools for dashboard_a):** Six tools (refresh_slot, set_rotation, set_brightness, wake, sleep, ota) node-addressed via logical name `dashboard_a`. Resources: `fleet_status`, `node_logs`.
- **AR20 (Node ID):** `health_<6 hex lowercase>` derived from MAC via `esp_read_mac(ESP_MAC_WIFI_STA, ...)`.
- **AR21 (Inter font set):** Pre-rasterised at 56, 44, 36, 22, 14, 12, 10 px via `lv_font_conv`; committed to `spine/assets/fonts/`.
- **AR22 (Diagnostics):** `diag_manager` emits over ESP-NOW events (`0x80+`) to master, forwarded to MCP as `{event:"node_log",...}`.
- **AR23 (Logger / secret redaction):** `LOG_SAFE()` macro strips known secret keys before emit (mechanical NFR-S6 enforcement).
- **AR24 (Offline queue):** Master buffers commands for sleeping targets; flushes on wake-ack.
- **AR25 (OTA streamer / manager):** Master chunks firmware binary; target `ota_manager` receives chunks, verifies SHA-256, writes to inactive partition, boots with rollback on failure.

### UX Design Requirements

Extracted from `ux-design-specification.md`. Each is a concrete build item.

- **UX-DR1 (Design token file):** `ui_tokens/olaf_tokens.h` — single source of truth for hex palette, ring geometry (r_outer=118, r_inner=106, centre 120,120, 12 px band), type scale, spacing, motion durations. No literals elsewhere.
- **UX-DR2 (Colour palette):** Exact hex values: BG `#000000`, FG `#F5F5F5`, muted `#8A8A8A`, dim `#555555`, status Good `#30D158`, Warning `#FFB020`, Bad `#FF3B30`, Unknown `#6E6E6E`, plus category accent colours per UX spec.
- **UX-DR3 (`olaf_dark` LVGL theme):** Custom theme module styling `lv_arc`, `lv_label`, `lv_chart`, `lv_obj`, `lv_anim`. All style overrides live in theme — no inline styles in compositions.
- **UX-DR4 (Inter font pipeline):** `spine/assets/fonts/` contains pre-rasterised Inter binaries at 56/44/36/22/14/12/10 px; compiled into firmware or loaded from SPIFFS.
- **UX-DR5 (StatusRing composition):** `lv_arc` 360° full, 12 px band; four states (Good/Warning/Bad/Unknown) with 80 ms ease-out colour tween.
- **UX-DR6 (HeadlineNumeral composition):** `lv_label` Inter Bold 56 px default, tabular numerals, variants default/secondary (44 px) / small-secondary (36 px) / compact (48 px); 200 ms ease-out value tween.
- **UX-DR7 (SecondaryNumeral composition):** HeadlineNumeral variant used on Round B, typically with category-accent tint.
- **UX-DR8 (CategoryLabel composition):** `lv_label` Inter Medium 12 px, all-caps, +0.04em tracking, muted, upper zone (y≈40–70).
- **UX-DR9 (Timestamp composition):** `lv_label` Inter Regular 10 px muted, lower zone (y≈180–200), string pushed by OLAF (node never computes).
- **UX-DR10 (RoundComposition):** Parent layout of StatusRing + CategoryLabel + Numeral + Timestamp inside 240×240, variants `primary` and `secondary`.
- **UX-DR11 (ChartTitle composition):** `lv_label` Inter Medium 14 px muted, top-left 16 px pad.
- **UX-DR12 (ChartCanvas composition):** `lv_chart` 208×240 px, no gridlines, no ticks, single-series accent stroke, three type variants (line / scatter / bar) with empty state "waiting for OLAF".
- **UX-DR13 (AxisExtremes composition):** Two `lv_label` Inter Regular 10 px dim; content pushed by OLAF.
- **UX-DR14 (RectangleComposition):** Parent layout of ChartTitle + ChartCanvas + AxisExtremes inside 240×320 portrait canvas.
- **UX-DR15 (CategoryRotator module):** Active-slot index + rotation timer (default 10 s, configurable); on expiry triggers coordinated cross-fade.
- **UX-DR16 (TransitionOrchestrator module):** Executes 300 ms synchronous cross-fade across all three displays as one event — no display acts alone.
- **UX-DR17 (StalenessMonitor module):** Tracks last-update timestamp per topic; dims ring or promotes to Unknown after configurable thresholds.
- **UX-DR18 (Three degraded visual states):** Fresh-boot placeholder (grey, "—", "waiting for OLAF"), stale-cached (dim rings, visible timestamp), live-current — three distinct states with explicit transition rules.
- **UX-DR19 (Three layers of attention):** Design honours peripheral (0.2–0.5 s @ 3–5 m), active (1–3 s @ 2 m), deep (5–10 s @ 1 m) legibility bands.
- **UX-DR20 (Motion timings):** Ring colour tween 80 ms ease-out; number tween 200 ms ease-out; category cross-fade 300 ms.

### FR Coverage Map

| FR | Epic(s) | Notes |
|---|---|---|
| FR1 — headline metric render | Epic 1, Epic 4 | Epic 1: dummy payload. Epic 4: live via REFRESH_SLOT. |
| FR2 — secondary metric render | Epic 1, Epic 4 | Same split. |
| FR3 — chart render | Epic 1, Epic 4 | Epic 1: static dataset. Epic 4: live. |
| FR4 — status ring render | Epic 1, Epic 4 | All four status tokens in Epic 1 via dummy state. |
| FR5 — timestamp/staleness indicator | Epic 1, Epic 4 | Epic 4 adds dim-on-stale. |
| FR6 — centre/ring render independently | Epic 1 | LVGL composition discipline. |
| FR7 — chart catalog (line, scatter, bar) | Epic 1, Epic 4 | Epic 1 proves all three variants; Epic 4 drives by payload. |
| FR8 — typography + palette per UX spec | Epic 1 | `olaf_dark` theme + `olaf_tokens.h`. |
| FR9 — N category slots in RAM | Epic 1, Epic 4 | Epic 1: `category_store` scaffold. Epic 4: wire-filled. |
| FR10 — slot-uniform (any N) | Epic 1, Epic 4 | Firmware treats all slots identically. |
| FR11 — rotation cadence | Epic 1, Epic 4 | Epic 1: fixed interval. Epic 4: ROTATION_CONFIG opcode. |
| FR12 — re-order rotation | Epic 4 | Via ROTATION_CONFIG. |
| FR13 — enable/disable categories | Epic 4 | Via ROTATION_CONFIG. |
| FR14 — pause rotation | Epic 4 | Via ROTATION_CONFIG. |
| FR15 — three-display simultaneous transition | Epic 1, Epic 4 | `TransitionOrchestrator`. |
| FR16 — instant transitions (MVP default) | Epic 1 | |
| FR17 — backlight brightness 0–255 | Epic 3, Epic 4 | Epic 3: first BRIGHTNESS demo. Epic 4: full. |
| FR18 — backlight 0 as idle state | Epic 1 | `backlight_controller`. |
| FR19 — directives apply immediately | Epic 4 | |
| FR20 — enter deep sleep on directive | Epic 4 | `sleep_manager` + SLEEP opcode. |
| FR21 — wake from deep sleep + re-render LKG | Epic 4 | Master re-push on wake-ack. |
| FR22 — no self-sleep | Epic 4 | |
| FR23 — ESP-NOW MAC whitelist | Epic 3 | Peer registration at boot. |
| FR24 — 2.4 GHz WiFi connect | Epic 2, Epic 4 | Epic 2: direct WiFi for OTA. Epic 4: on-demand via WIFI_UP. |
| FR25 — receive opcode envelopes | Epic 3, Epic 4 | Epic 3: one opcode. Epic 4: full set. |
| FR26 — auto-reconnect WiFi/ESP-NOW | Epic 4 | `util_backoff`. |
| FR27 — parse all opcodes | Epic 4 | `target_dispatcher` full. |
| FR28 — ignore unknown fields / status → grey | Epic 3, Epic 4 | Lenient parse established in Epic 3. |
| FR29 — ESP-NOW RX in deep sleep | Epic 3, Epic 4 | Epic 3: awake RX. Epic 4: RTC receiver. |
| FR30 — LKG cache in RAM | Epic 4 | `store_lkg`. |
| FR31 — render from LKG on link loss | Epic 4 | |
| FR32 — staleness indication (dim ring) | Epic 4 | `StalenessMonitor`. |
| FR33 — fresh-boot "waiting for OLAF" placeholder | Epic 1 | No fabricated data. |
| FR34 — value + timestamp + status as one unit | Epic 1 | Composition rule. |
| FR35 — provision WiFi + master MAC | Epic 2 | `make factory-flash`. |
| FR36 — operator-triggered OTA | Epic 2, Epic 4, Epic 5 | Epic 2: direct. Epic 4: master-streamed. Epic 5: MCP-tool-triggered. |
| FR37 — download, verify, flash, rollback | Epic 2 | Dual-partition + SHA-256. |
| FR38 — "updating" visual state | Epic 2 | |
| FR39 — clean first boot | Epic 1 | |
| FR40 — online/offline status publish | Epic 3, Epic 4 | Epic 3: link-state skeleton. Epic 4: full LWT via keepalive. |
| FR41 — firmware version on boot | Epic 2 | |

All 41 FRs mapped. No orphans.

## Epic List

### Epic 1: Target Display Platform & Honest-Mirror Rendering *(Milestone 1)*

Deliver a physical Node 01 that renders the full UX-spec appearance on all three displays using hard-coded dummy payloads, with zero networking. This validates the rendering stack, the `olaf_dark` theme + `olaf_tokens.h` layer, and all ten LVGL compositions before any wire protocol exists. Exit gate: the dashboard passes the "would this survive on the wall" aesthetic test on dummy data.

**FRs covered:** FR1, FR2, FR3, FR4, FR5, FR6, FR7, FR8, FR9, FR10 (partial), FR11 (hardcoded cadence), FR15, FR16, FR18, FR33, FR34, FR39
**NFRs covered:** NFR-P5, NFR-Pw1, NFR-Pw2, NFR-Pw3, NFR-Pw5, NFR-Pw6, NFR-U1, NFR-U2, NFR-U3, NFR-U4, NFR-M1, NFR-M2, NFR-M3, NFR-M4
**Additional + UX:** AR1, AR2 (partial), AR3, AR4, AR5, AR11 (UI subset), AR21; UX-DR1–UX-DR20

### Epic 2: Target OTA Pipeline & Factory Provisioning *(Milestone 1)*

Deliver an operational OTA path on the target: dual-partition `app0`/`app1` layout, SHA-256 verification, rollback on failed boot, a visually distinct "updating" state, and a `make factory-flash` one-shot serial channel that pre-writes WiFi + master MAC into NVS. OTA trigger in this epic is direct WiFi pull — the master-streamed path supersedes it in Epic 4. Firmware version reports on boot.

**FRs covered:** FR24 (WiFi for OTA), FR35, FR36 (direct-pull path), FR37, FR38, FR41
**NFRs covered:** NFR-R5, NFR-S1, NFR-S3, NFR-S5, NFR-S6
**Additional:** AR2, AR6, AR7, AR20, AR23, AR25 (target-side)

### Epic 3: Master + First ESP-NOW Link *(Milestone 2)*

Stand up the master (ESP32-C3 Mini): PlatformIO scaffold, persistent WiFi, ESP-NOW initialization with MAC whitelist + AES-CCM encryption, routing table bootstrap, and envelope pack/unpack. Demo gate: master sends a single BRIGHTNESS opcode over ESP-NOW and the target acts on it. Two boxes talk.

**FRs covered:** FR17 (over-the-air brightness), FR23, FR25 (one opcode), FR28 (lenient parse), FR29 (awake RX), FR40 (online/offline detect skeleton)
**NFRs covered:** NFR-I4 (envelope + WS stub), NFR-I5, NFR-P4, NFR-S4
**Additional:** AR9 (`net_espnow`, `util_json`), AR10 (`routing_table`, `ws_mcp_endpoint` stub), AR12, AR14, AR15

### Epic 4: Full Target Protocol & Degraded-State Rendering *(Milestone 3)*

Complete the target's wire-driven behaviour: full opcode set (WAKE, SLEEP, REFRESH_SLOT, ROTATION_CONFIG, BRIGHTNESS, WIFI_UP, WIFI_DOWN, OTA_BEGIN, DIAG_REQUEST + events), RTC ESP-NOW receiver in deep sleep, wake-to-first-render ≤3 s, on-demand WiFi TCP for large payloads, master-streamed OTA (supersedes Epic 2's direct path), LKG cache + StalenessMonitor, three degraded visual states (fresh / stale / live), coordinated rotation driven over the wire, auto-reconnect, offline queue on master, ESP-NOW fragmentation for >250 B payloads, and diagnostic events. PRD journeys 1–6 work end-to-end short of the MCP agent.

**FRs covered:** FR10 (live slots), FR11 (wire-configurable), FR12, FR13, FR14, FR15 (live coordinated), FR17 (full), FR19, FR20, FR21, FR22, FR26, FR27 (all opcodes), FR30, FR31, FR32, FR36 (master-streamed), FR40 (full LWT)
**NFRs covered:** NFR-P1, NFR-P2, NFR-P3, NFR-P4, NFR-P6, NFR-R1, NFR-R2, NFR-R3, NFR-R4, NFR-R6, NFR-Pw4, NFR-I1, NFR-I2, NFR-I3, NFR-I6
**Additional + UX:** AR9 (`net_espnow_frag`, `net_wifi_lifecycle`, `net_tcp_server`, `net_tcp_client`, `store_lkg`, `ota_manager`, `sleep_manager`, `diag_manager`), AR10 (`transport_selector`, `offline_queue`, `ota_streamer` full), AR11 (full), AR13, AR16, AR22, AR24, AR25 (full); UX-DR15, UX-DR17, UX-DR18

### Epic 5: MCP Agent Surface *(Milestone 4)*

Expose the fleet to MCP-capable agents (OLAF, Claude, scripts). Python MCP server (HTTP/SSE) with WebSocket client to master; master's `ws_mcp_endpoint` as WebSocket server on `:8080`; six tools for `dashboard_a` (refresh_slot, set_rotation, set_brightness, wake, sleep, ota); `fleet_status` and `node_logs` resources; logical-name → MAC routing so agents never see MACs; `systemd` service. Exit gate: Journey 4 passes end-to-end — OLAF pushes a category the firmware has never heard of, and it renders with no rebuild.

**FRs covered:** FR36 (agent-triggered OTA); all others validated via agent-directed path
**NFRs covered:** NFR-I4 (full), NFR-I6
**Additional:** AR10 (`ws_mcp_endpoint` full), AR17, AR18, AR19

## Epic 1: Target Display Platform & Honest-Mirror Rendering

Deliver a physical Node 01 that renders the full UX-spec appearance on all three displays using hard-coded dummy payloads, with zero networking. Proves the rendering stack, the `olaf_dark` theme, the token layer, and the ten LVGL compositions before any wire protocol exists.

### Story 1.1: Faceplate enclosure — Onshape design and 3D print

As **Kamal**,
I want **the Node 01 faceplate designed in Onshape and a printed part in hand**,
So that **every downstream hardware story (wiring, assembly, on-wall gate) builds against a committed physical reference.**

**Acceptance Criteria:**

**Given** the PRD hardware envelope (2× GC9A01 2.28" round + 1× ILI9341 2.8" in portrait, OLED-deep-black faceplate, plastic/acrylic only — no metal)
**When** the faceplate model is published in Onshape
**Then** the display cutouts and mounting bosses match the panel drawings with clearance for bezels and ribbon cables
**And** the layout places the ILI9341 on the left in portrait and the two GC9A01 rounds stacked vertically on the right, centred along the rectangle's mid-line (per UX spec faceplate anatomy)
**And** there are internal standoffs or slots that secure the ESP32-S3-WROOM-1 DevKit without requiring metal fasteners near the PCB antenna

**Given** a public Onshape document link committed to `nodes/node-01-health-dashboard/hardware/3d-print/README.md`
**When** opened in a browser
**Then** the document is viewable (public read link), contains the part studio, and lists print settings (material, layer height, infill) in the document description

**Given** the part printed at the committed settings
**When** the three displays are dry-fitted (no wiring yet)
**Then** all three seat flush, no cutout is oversize enough to reveal backlight leak, and the DevKit fits without forcing

**Given** exported STL and STEP files committed to `nodes/node-01-health-dashboard/hardware/3d-print/`
**When** another operator opens the folder
**Then** they can reprint the part without opening Onshape

### Story 1.2: Wiring harness assembly and wiring guide

As **Kamal**,
I want **all three displays wired to the ESP32-S3 DevKit on a shared SPI bus per a committed wiring guide**,
So that **Story 1.4 (three-display LVGL init) has a working harness to drive, and a future rebuild is reproducible.**

**Acceptance Criteria:**

**Given** the GPIO budget from the PRD (shared MOSI, SCLK, BL, RST + per-display CS×3, DC×3 — 10 signal pins total) and the ESP32-S3-WROOM-1 N16R8 DevKit pinout
**When** `nodes/node-01-health-dashboard/hardware/wiring/wiring-guide.md` is committed
**Then** it contains a locked pin map (one GPIO per signal, avoiding strap pins), a full harness diagram (MOSI/SCLK fan-out, VCC/GND fan-out via JST splitters, per-display CS/DC/RST), and a labelled photo of the completed harness
**And** it includes a continuity checklist (every pin → every destination) the operator can tick through before power-on

**Given** the pin map in the wiring guide
**When** referenced from `components/` (e.g., `main/board_pins.h` or equivalent)
**Then** the firmware pin definitions match the guide one-for-one (single source of truth — guide updates require a firmware update and vice versa)

**Given** the harness assembled and first power-on applied at 5 V via USB-C
**When** current draw is measured at idle (no firmware driving displays yet)
**Then** cold-start inrush does not trip the USB-C source (no brown-out, no hub protection)

**Given** the wired assembly
**When** a simple ESP-IDF "hello SPI" test (bench utility, not product firmware) writes a known command to each panel in turn
**Then** each panel responds (ID register read or backlight flash), confirming all three CS lines and shared MOSI/SCLK are wired correctly

### Story 1.3: PlatformIO + ESP-IDF 5.5 monorepo scaffold

As **Operator (Kamal)**,
I want **a reproducible PlatformIO + ESP-IDF 5.5 project scaffold for the target node inside the spine/nodes monorepo**,
So that **every subsequent story builds on the same committed toolchain with no environment drift.**

**Acceptance Criteria:**

**Given** a fresh clone of the repository
**When** I run `pio run -e olaf_healthdash_node1` in `nodes/node-01-health-dashboard/firmware/`
**Then** the firmware builds successfully against ESP-IDF 5.5 and `esp32-s3-devkitc-1` board config
**And** `platformio.ini`, `partitions.csv`, `main/idf_component.yml` match the stanzas committed in `documents/planning-artifacts/architecture.md`
**And** `idf_component.yml` declares `lvgl/lvgl ~9.5.0`, `espressif/esp_lvgl_port`, `espressif/esp_lcd_gc9a01`, `espressif/esp_lcd_ili9341`
**And** the repository root contains `spine/`, `nodes/node-00-master/`, `nodes/node-01-health-dashboard/`, `mcp/`, and `docs/` directories (may be empty scaffolds)

**Given** a built binary flashed to the dev board
**When** the board boots
**Then** the serial console at 115200 baud emits an ESP-IDF boot banner and a single log line identifying the node (e.g., `health_<mac6hex>`)

### Story 1.4: Three-display SPI bus and LVGL multi-display initialization

As **the Node firmware**,
I want **all three displays registered with LVGL as independent `lv_disp_t` instances over a shared SPI bus**,
So that **every composition in the UX spec has a canvas to render into.**

**Acceptance Criteria:**

**Given** wired hardware (2× GC9A01 + 1× ILI9341 on a shared SPI bus with per-display CS/DC and shared MOSI/SCLK/RST/BL)
**When** the firmware boots
**Then** `esp_lcd_gc9a01` drives both round panels and `esp_lcd_ili9341` drives the rectangle panel in 90° portrait orientation
**And** `esp_lvgl_port` registers three `lv_disp_t` instances, each with a double-buffer in PSRAM sized for its display
**And** a ~4 KB DMA ping-pong buffer per display lives in internal SRAM
**And** a single LVGL render task is pinned to core 1; WiFi/application work stays on core 0

**Given** all three displays initialised
**When** the firmware paints a solid `#FF00FF` fill on each
**Then** each display shows magenta with no tearing, no staggered paint across the trio

### Story 1.5: Design tokens and `olaf_dark` LVGL theme

As **the Node firmware**,
I want **a single source of truth for visual tokens and an LVGL theme that applies them to every widget**,
So that **no composition hardcodes colours, sizes, or geometry — enforcing the UX spec mechanically.**

**Acceptance Criteria:**

**Given** the UX Design Specification hex palette, ring geometry, type scale, spacing, and motion values
**When** `components/ui_tokens/olaf_tokens.h` is committed
**Then** every value from the UX spec (colours, ring r_outer=118, r_inner=106, centre 120,120, band 12 px, type sizes 56/44/36/22/14/12/10, motion 80/200/300 ms) is defined as a named macro or constant
**And** no hex literal or numeric geometry constant appears in any other source file (verified by grep)

**Given** `components/ui_theme/olaf_dark.c` is committed
**When** the theme is registered at LVGL init
**Then** styling for `lv_arc`, `lv_label`, `lv_chart`, `lv_obj`, and `lv_anim` resolves through the theme
**And** backgrounds on all three displays render at the OLED-deep-black value defined in `olaf_tokens.h`

**Given** pre-rasterised Inter fonts committed under `spine/assets/fonts/` at 56, 44, 36, 22, 14, 12, and 10 px
**When** a test `lv_label` is created at 56 px
**Then** it renders in Inter Bold with tabular numerals

### Story 1.6: Round display compositions (StatusRing, Numerals, CategoryLabel, Timestamp, RoundComposition)

As **the Node firmware**,
I want **all round-display compositions implemented as named LVGL templates driven by payload parameters**,
So that **each round can render a headline or secondary metric identically across categories.**

**Acceptance Criteria:**

**Given** `components/ui_composition_round/` is committed
**When** a RoundComposition is instantiated on a GC9A01 display
**Then** it contains a StatusRing (`lv_arc`, full 360°, 12 px band), a HeadlineNumeral or SecondaryNumeral (`lv_label` Inter, variants default/secondary/small-secondary/compact), a CategoryLabel (`lv_label` Inter Medium 12 px upper zone), and a Timestamp (`lv_label` Inter Regular 10 px lower zone)

**Given** a RoundComposition rendering a dummy payload `{value: "84.1", unit: "kg", label: "Body weight", timestamp_text: "this morning", status: "good"}`
**When** the payload `status` changes to `warn`
**Then** the StatusRing colour tweens from `#30D158` to `#FFB020` over 80 ms ease-out
**And** the centre numeral and timestamp do not repaint (independent-render rule)

**Given** a RoundComposition with `variant=secondary`
**When** it renders
**Then** the SecondaryNumeral uses the category's accent colour rather than foreground white

**Given** a RoundComposition numeric value changing from `84.1` to `83.8`
**When** the payload arrives
**Then** the HeadlineNumeral tweens between old and new values over 200 ms ease-out

### Story 1.7: Rectangle chart compositions (ChartTitle, ChartCanvas, AxisExtremes, RectangleComposition)

As **the Node firmware**,
I want **the chart compositions implemented for line, scatter, and bar variants with an empty state**,
So that **the rectangle display can render any category's chart driven by payload.**

**Acceptance Criteria:**

**Given** `components/ui_composition_rect/` is committed
**When** a RectangleComposition is instantiated on the ILI9341 in portrait orientation
**Then** it contains a ChartTitle (`lv_label` Inter Medium 14 px muted, top-left 16 px pad), a ChartCanvas (`lv_chart` 208×240 px area, no gridlines, no ticks), and two AxisExtremes labels (`lv_label` Inter Regular 10 px dim)

**Given** a ChartCanvas rendering with `type="line"`
**When** a dummy 30-day dataset is passed
**Then** a single polyline is drawn with 2 px stroke and rounded joins in the category accent colour

**Given** a ChartCanvas rendering with `type="scatter"`
**When** a dummy glucose dataset is passed
**Then** points render at 3 px radius in the category accent colour

**Given** a ChartCanvas rendering with `type="bar"`
**When** a dummy 7-day dataset is passed
**Then** vertical bars render with 4 px gap and accent fill

**Given** a ChartCanvas with no data
**When** it is asked to render
**Then** it shows a subtle dim placeholder text "waiting for OLAF"

### Story 1.8: Category store and fresh-boot placeholder state

As **the Node firmware**,
I want **a uniform N-slot category store and a fresh-boot placeholder state**,
So that **the firmware renders honestly when no content has been pushed yet, and treats every category slot identically.**

**Acceptance Criteria:**

**Given** `components/category_store/` is committed
**When** the store is initialised
**Then** it exposes `get(slot_id)` and `set(slot_id, kind, payload)` for any slot_id between 1 and a configured maximum (default 16)
**And** no category-specific string ("glucose", "weight", "cycling", "fasting") appears anywhere in the store source (verified by grep)

**Given** the firmware boots and the category store is empty
**When** LVGL begins rendering
**Then** all three displays show the fresh-boot placeholder: grey StatusRing, "—" or blank where values would be, "waiting for OLAF" text on the rectangle, no fabricated numbers, no "hello world" test data

**Given** a hard-coded test fixture at boot (`main/dummy_fixtures.c`, behind a build flag) populates four dummy slots
**When** rendering resumes
**Then** each slot's headline, secondary, and chart payloads render on the appropriate displays when that slot becomes active

### Story 1.9: Rotation timer and coordinated three-display cross-fade

As **the Viewer**,
I want **all three displays to change to the next category at the same instant**,
So that **the dashboard presents one coherent snapshot at every moment, not three independent widgets.**

**Acceptance Criteria:**

**Given** `components/category_rotator/` with a fixed 10 s interval and at least two populated slots
**When** the rotation timer expires
**Then** `components/render_transition/` triggers a 300 ms cross-fade on all three `lv_disp_t` instances beginning within the same LVGL tick

**Given** a 240 fps slow-motion video of a rotation event
**When** reviewed frame-by-frame
**Then** no display begins its cross-fade more than ~50 ms before or after any other (a mechanical anchor for NFR-P3 verified in Epic 4 gate)

**Given** rotation active across four dummy slots
**When** observed for 60 s
**Then** the active category advances round-robin through all enabled slots with no visible stagger and no torn frames

### Story 1.10: Backlight controller and backlight-0 idle state

As **the Operator**,
I want **PWM backlight control across all three displays with backlight-0 as a first-class idle state**,
So that **brightness can be validated on the bench before any wire protocol exists, and backlight-0 idle holds the renderer running.**

**Acceptance Criteria:**

**Given** `components/backlight_controller/` is committed
**When** it initialises
**Then** the shared BL GPIO is driven as 8-bit PWM (0–255) via `ledcWrite()` or the ESP-IDF LEDC driver

**Given** a debug serial command `bl <0-255>` (behind a dev build flag)
**When** the operator issues `bl 0`
**Then** all three displays go visually dark while the LVGL render task continues to tick and log frames (backlight-0 = idle, not off)
**And** a multimeter at the USB-C input reads ~100 mA ±10%

**Given** `bl 255`
**When** measured on the bench with all compositions rendering
**Then** current draw is ≤ 420 mA at 5 V

**Given** `bl 128`
**When** measured on the bench
**Then** current draw is ≤ 220 mA at 5 V

### Story 1.11: On-wall aesthetic exit gate

As **Kamal**,
I want **a recorded validation that the assembled node passes the aesthetic bar on dummy data**,
So that **Milestone 1 is proven before we add the distraction of networking in Milestone 2.**

**Acceptance Criteria:**

**Given** a node fully assembled inside the 3D-printed faceplate enclosure, USB-C powered, running the dummy-fixtures build
**When** viewed from 3–5 m across the room
**Then** the primary metric value and status ring colour are readable without squinting (Layer 1 peripheral glance)

**Given** the same node viewed from 2 m
**When** observed for 1–3 s
**Then** the category label, secondary metric, and chart shape are all legible (Layer 2 active glance)

**Given** the same node viewed from 1 m
**When** observed for 5–10 s
**Then** the chart detail, axis extremes, and timestamp are readable (Layer 3 deep glance)

**Given** a 240 fps video of rotation over a 60 s window
**When** reviewed
**Then** transitions across the three displays look coordinated (informal Epic 1 check; NFR-P3 measured formally in Epic 4 gate)

**Given** the node running for 15 min on a wall mount
**When** walked past at normal pace
**Then** the result passes the "would this survive in Kamal's hallway / Apple showroom" gut check (recorded as a journal entry with photos)

## Epic 2: Target OTA Pipeline & Factory Provisioning

Deliver an operational OTA path on the target plus a one-shot factory-flash channel that writes WiFi credentials and the master MAC into NVS. Dual-partition `app0`/`app1` layout, SHA-256 verification, rollback on failed boot, a distinct "updating" visual state. OTA trigger in this epic is a direct WiFi pull — the master-streamed path supersedes it in Epic 4. Firmware version reports on boot.

### Story 2.1: Factory-flash NVS provisioning channel

As **Kamal (Operator)**,
I want **a `make factory-flash` one-shot serial channel that writes WiFi credentials and the master MAC into NVS**,
So that **every target ships with provisioning baked in, and no runtime re-provisioning path is ever needed.**

**Acceptance Criteria:**

**Given** a target connected via USB-C with `factory/credentials.env` supplied by the operator
**When** `make factory-flash` is invoked from `nodes/node-01-health-dashboard/`
**Then** the script writes `wifi/ssid`, `wifi/pwd`, `olaf/mac` (master MAC) into the `nvs` partition and optionally `mqtt/host`, `mqtt/port` placeholders (reserved, unused in v2)
**And** the committed `factory/credentials.env.example` documents every key without leaking real secrets

**Given** a target that has been factory-flashed
**When** the firmware boots
**Then** it reads all NVS keys at startup; any missing required key causes the fresh-boot placeholder state to display with a single serial log line naming the missing key

**Given** the factory-flash path
**When** inspected
**Then** it is compiled into a **factory build** only — the production build path (set by a build flag) contains no code path that accepts WiFi credentials or master MAC over serial or any other runtime channel

### Story 2.2: Target WiFi client for OTA transport (persistent variant)

As **the Node firmware**,
I want **a WiFi station client that connects using NVS credentials and auto-reconnects on drop**,
So that **OTA has a transport in Epic 2, and Epic 4 can later restyle this into the on-demand lifecycle.**

**Acceptance Criteria:**

**Given** factory-flashed NVS credentials and `spine/components/net_wifi_lifecycle/` (or an interim `net_wifi_persistent_target/` used only in Epic 2 and retired in Epic 4)
**When** the firmware boots
**Then** it associates with the configured 2.4 GHz SSID within ≤ 20 s and logs RSSI once associated

**Given** an associated target
**When** the access point drops and recovers
**Then** the client reconnects automatically within ≤ 60 s of network restoration without operator intervention

**Given** malformed or incorrect credentials in NVS
**When** association fails repeatedly (configurable threshold, default 3 attempts)
**Then** the firmware logs a single honest error line, leaves the fresh-boot placeholder rendered, and does not crash or tight-loop

### Story 2.3: Dual-partition OTA download, SHA-256 verify, and rollback

As **Kamal (Operator)**,
I want **OTA firmware updates that download, verify, activate, and self-rollback on failed boot**,
So that **a bad build can never brick a node on the wall.**

**Acceptance Criteria:**

**Given** `partitions.csv` committed with `app0` (ota_0, 4 MB) + `app1` (ota_1, 4 MB) + `otadata` + `coredump` + `spiffs` per the architecture
**When** the firmware flashes for the first time
**Then** it runs from `app0` and `app1` is empty; `esp_ota_get_running_partition()` returns `app0`

**Given** `spine/components/ota_manager/` exposes a `start_ota(url, expected_sha256)` entry point
**When** invoked with a valid URL to a firmware binary
**Then** the manager downloads the binary via HTTP pull into the inactive partition, computes SHA-256, and compares against `expected_sha256`
**And** on match, calls `esp_ota_set_boot_partition()` on the inactive partition and triggers reboot
**And** on mismatch, aborts the flash, retains the running firmware, and emits a single honest error log

**Given** a deliberately broken binary (e.g., corrupted after verify — simulated by flipping a byte in `app1` post-activate)
**When** the node reboots
**Then** ESP-IDF's bootloader rollback brings `app0` back as the active partition and the node boots cleanly on the old firmware

**Given** a power cut mid-download
**When** the node power-cycles
**Then** the running partition is untouched and the node boots normally; partial bytes in the inactive partition are irrelevant because `otadata` was not updated

### Story 2.4: "Updating" visual state during OTA download and flash

As **the Viewer**,
I want **a distinct "updating" screen during OTA so the dashboard never goes blank or looks crashed**,
So that **a mid-update glance during the 5–15 second flash window is honest instead of alarming.**

**Acceptance Criteria:**

**Given** `components/render_lifecycle/` exposes an `enter_updating_state()` call
**When** invoked at OTA start
**Then** all three displays switch to the updating state: OLED-deep-black background, a subtle progress indicator (e.g., pulsing dot or thin arc), and a small "updating" label in muted colour per the UX spec token palette

**Given** the updating state active during download + flash
**When** observed
**Then** the display never goes fully blank and never shows a crash / garbage / white-screen state

**Given** OTA completes successfully and the node reboots into the new image
**When** the fresh firmware starts
**Then** the fresh-boot placeholder state renders (no persisted LKG in M1 — that comes in Epic 4)

**Given** OTA aborts (hash mismatch, download failure)
**When** the abort occurs
**Then** the updating state exits cleanly back to whatever was rendering before OTA started — the running firmware has not restarted

### Story 2.5: Firmware version publish on boot

As **Kamal (Operator)**,
I want **every boot to emit a clear firmware version line**,
So that **I can confirm an OTA actually landed without a multimeter or a logic analyser.**

**Acceptance Criteria:**

**Given** a build tag defined in `platformio.ini` (e.g., via `-DFW_VERSION="0.1.3"` or derived from `git describe`)
**When** the firmware boots
**Then** the serial console at 115200 baud logs a single line in the form `fw_version=<version> node_id=health_<mac6hex>` within the first second of boot
**And** the version string is also stored in RAM so Epic 3's diag channel and Epic 4's event stream can emit it over ESP-NOW (wiring happens in those epics, not here)

**Given** two builds with different `FW_VERSION` values
**When** OTA swaps one for the other and the node reboots
**Then** the first serial line after reboot shows the new version — confirming the update landed

### Story 2.6: Secret redaction in logs via `LOG_SAFE`

As **the Node firmware**,
I want **a mechanical `LOG_SAFE()` macro that strips known secret keys before emit**,
So that **NFR-S6 is enforced by code, not by developer discipline.**

**Acceptance Criteria:**

**Given** a `LOG_SAFE()` macro committed in the spine logger module
**When** any source file logs a structure containing NVS keys `wifi/pwd`, `mqtt/user`, `mqtt/pwd`, or future secret-tagged keys
**Then** the emitted log line shows the key name but replaces the value with `***REDACTED***`

**Given** the entire codebase grepped for raw `printf` / `ESP_LOGI` / `ESP_LOGE` calls that reference known secret key names
**When** the build-time lint runs (or a commit-hook grep)
**Then** zero matches outside `LOG_SAFE()` wrappings are found (enforcement is mechanical, not stylistic)

**Given** a factory build and a production build
**When** `idf.py monitor` is attached after boot and the operator provokes a WiFi auth failure
**Then** neither build ever emits the raw WiFi password in any log line

## Epic 3: Master + First ESP-NOW Link

Stand up the master node (ESP32-C3 Mini) and prove the dual-radio spine end-to-end: master joins WiFi, talks ESP-NOW to the target, the target dispatches opcodes. Exit gate is a single BRIGHTNESS opcode flowing master → target and the target backlight changing. Two boxes talk.

### Story 3.1: Master PlatformIO scaffold and factory-flash

As **Kamal (Operator)**,
I want **a committed PlatformIO + ESP-IDF 5.5 scaffold for the master node (ESP32-C3 Mini) with its own `make factory-flash` channel**,
So that **subsequent stories have a reproducible master build and provisioned NVS to work against.**

**Acceptance Criteria:**

**Given** `nodes/node-00-master/firmware/platformio.ini` committed for `board = esp32-c3-devkitm-1` (or equivalent C3 Mini board config) with `framework = espidf` and ESP-IDF ≥ 5.5
**When** `pio run -e olaf_master` is invoked from `nodes/node-00-master/firmware/`
**Then** the master firmware builds cleanly against ESP-IDF 5.5

**Given** `nodes/node-00-master/firmware/partitions.csv` committed (simpler layout than target — single app partition acceptable since master is not OTA'd in this epic)
**When** flashed to an ESP32-C3 Mini
**Then** the board boots and logs `node_id=master_<mac6hex>` on the serial console within 1 s of boot

**Given** `nodes/node-00-master/factory/credentials.env.example` committed documenting keys `wifi/ssid`, `wifi/pwd`, and `targets/dashboard_a/mac`, `targets/dashboard_a/key` (AES-CCM per-peer key)
**When** `make factory-flash` is invoked with the real `credentials.env`
**Then** the script writes those keys into the master's NVS partition
**And** no runtime-provisioning code path exists in the production build

### Story 3.2: Master persistent WiFi client

As **the Master firmware**,
I want **a WiFi station client that connects at boot and stays connected for the lifetime of the node**,
So that **the MCP WebSocket endpoint (Epic 5) has a transport and Epic 3's link-state logic has something to ride on.**

**Acceptance Criteria:**

**Given** `spine/components/net_wifi_persistent/` committed (master-only variant)
**When** the master boots with factory-flashed credentials
**Then** it associates with the 2.4 GHz SSID within ≤ 20 s and logs RSSI once associated

**Given** an associated master
**When** the access point drops and recovers
**Then** the master reconnects automatically within ≤ 60 s using `util_backoff` exponential backoff

**Given** the master running for ≥ 1 hour in a healthy network
**When** observed
**Then** no reconnection loop, no memory leak, and RSSI is logged once per minute at DEBUG level

### Story 3.3: ESP-NOW envelope pack/unpack (`util_json`)

As **the spine codebase**,
I want **a committed envelope pack/unpack utility shared by master and target**,
So that **wire format is defined in one place, symmetrically usable on both sides, and unit-tested offline.**

**Acceptance Criteria:**

**Given** `spine/components/util_json/` committed with `espnow_envelope_pack()` and `espnow_envelope_unpack()` functions
**When** called with `{magic=0xOLAF, opcode=0x05 BRIGHTNESS, seq=0, body="{\"level\":128}"}`
**Then** the output is a contiguous byte buffer matching the format in `docs/espnow-wire-protocol.md`: `2 B magic + 1 B opcode + 2 B seq/fr + 2 B body_len + body`

**Given** a packed buffer from the above
**When** `espnow_envelope_unpack()` is called on it
**Then** it returns the original opcode, seq, and body bytes without loss

**Given** a buffer with `magic` mismatched, `body_len > buffer_size`, or zero length
**When** unpack is called
**Then** it returns a non-zero error code and does not crash, segfault, or read past the buffer

**Given** Unity unit tests committed under `spine/components/util_json/test/`
**When** `idf.py test` runs on-device or under QEMU
**Then** round-trip pack → unpack succeeds for BRIGHTNESS, REFRESH_SLOT, and a fragmented (seq/fr-encoded) body

### Story 3.4: ESP-NOW initialization, MAC whitelist, and AES-CCM encryption

As **the Node firmware (both master and target)**,
I want **ESP-NOW initialised with encrypted peering against a MAC whitelist**,
So that **only the expected counterpart can send or receive, and unexpected RF chatter is silently dropped.**

**Acceptance Criteria:**

**Given** `spine/components/net_espnow/` committed (shared by master and target builds)
**When** master initialises
**Then** it registers the target MAC (from NVS `targets/dashboard_a/mac`) as an encrypted peer with the per-peer AES-CCM key from `targets/dashboard_a/key`

**Given** the target initialises
**When** WiFi station is up (Epic 2 persistent client)
**Then** it registers the master MAC (from NVS `olaf/mac`) as an encrypted peer using the matching per-peer key
**And** incoming ESP-NOW frames from any MAC other than the whitelisted master are dropped before reaching `target_dispatcher`

**Given** both sides peered
**When** a master sends an envelope-packed `BRIGHTNESS` opcode to the target MAC
**Then** the target's `net_espnow` RX callback receives the decrypted payload within a single RF burst and hands it to `target_dispatcher`

**Given** a deliberately tampered frame (e.g., wrong peer key on master side — simulated by flipping a byte)
**When** received
**Then** ESP-NOW's built-in AES-CCM verification rejects it and the application layer sees no event

### Story 3.5: Routing table on master (logical name → MAC)

As **the Master firmware**,
I want **a routing table mapping logical names (e.g., `dashboard_a`) to MAC addresses**,
So that **higher-layer clients (MCP, debug serial) address targets by name and never see MACs.**

**Acceptance Criteria:**

**Given** `nodes/node-00-master/firmware/components/routing_table/` committed
**When** the master boots
**Then** it loads every `targets/<logical_name>/mac` NVS entry into an in-RAM routing table
**And** exposes `routing_lookup(logical_name) → mac` and `routing_list()` API calls

**Given** `nodes/node-00-master/firmware/components/transport_selector/` committed as a stub
**When** called with `(logical_name, opcode, body_size)`
**Then** it returns `TRANSPORT_ESPNOW` unconditionally in Epic 3 (ESP-NOW vs WiFi TCP decision logic lands in Epic 4)

**Given** a debug serial command on the master (e.g., `send dashboard_a brightness 64`)
**When** invoked
**Then** the master resolves `dashboard_a` via the routing table, packs a BRIGHTNESS envelope, and sends it via `net_espnow` to the resolved MAC

### Story 3.6: Target dispatcher skeleton and BRIGHTNESS handler

As **the Target firmware**,
I want **a dispatcher that maps incoming opcodes to handlers, starting with BRIGHTNESS**,
So that **Epic 3's end-to-end demo works and Epic 4 can bolt on remaining opcodes without restructuring.**

**Acceptance Criteria:**

**Given** `nodes/node-01-health-dashboard/firmware/components/target_dispatcher/` committed
**When** the target receives a decrypted envelope from `net_espnow`
**Then** the dispatcher validates the envelope, extracts the opcode, and routes to a registered handler

**Given** a BRIGHTNESS handler registered in the dispatcher
**When** a `{"level": 128}` body arrives
**Then** the handler parses the JSON body, clamps to 0–255, and calls `backlight_controller_set(128)`
**And** the handler applies the change within ≤ 500 ms of envelope receipt (NFR-P4)

**Given** an envelope with an opcode the dispatcher does not recognise (e.g., `0x77`)
**When** received
**Then** the dispatcher logs a single honest line at DEBUG and drops the envelope without crash or state change (lenient-parse — NFR-I3)

**Given** an envelope body that fails JSON parse (truncated, malformed)
**When** received
**Then** the dispatcher logs the failure via `LOG_SAFE()` and drops the envelope; last-known-good backlight is retained (NFR-R6 foundation, fully realised in Epic 4 LKG)

**Given** the full path master → ESP-NOW → target dispatcher → backlight
**When** an operator sends `send dashboard_a brightness 0` from the master serial debug command
**Then** the target's backlight goes to 0 within ≤ 500 ms while LVGL rendering continues to tick
**And** sending `send dashboard_a brightness 255` brings it back to full brightness within the same budget

### Story 3.7: Link-state detection skeleton (online / offline)

As **the Master firmware**,
I want **awareness of whether each peered target is currently reachable**,
So that **Epic 4's offline queue and Epic 5's `fleet_status` resource have a foundation to build on.**

**Acceptance Criteria:**

**Given** every outbound ESP-NOW send records a timestamp and awaits the MAC-layer ACK (ESP-NOW built-in)
**When** the ACK arrives within the timeout (default 100 ms)
**Then** the routing table entry for that target records `last_seen_ms = now()` and `state = online`

**Given** a target whose last ACK is older than a configurable threshold (default 60 s)
**When** a periodic keepalive check ticks (default every 10 s)
**Then** the target's state transitions to `offline` and a single serial log line records the transition

**Given** an offline target that subsequently ACKs a send
**When** the ACK arrives
**Then** the state transitions back to `online` and a single serial log line records the recovery

**Given** the master's debug serial `status` command
**When** invoked
**Then** it prints every target in the routing table with logical name, MAC, current state, and last-seen timestamp

## Epic 4: Full Target Protocol & Degraded-State Rendering

Complete the target's wire-driven behaviour: full opcode set, deep sleep + RTC wake, on-demand WiFi TCP, master-streamed OTA (supersedes Epic 2's direct path), LKG cache + StalenessMonitor, three degraded visual states, coordinated rotation driven over the wire, auto-reconnect, offline queue on master, ESP-NOW fragmentation, diagnostic events. PRD journeys 1–6 work end-to-end short of the MCP agent. All performance NFRs gated here.

### Story 4.1: Full opcode dispatcher (REFRESH_SLOT, ROTATION_CONFIG, DIAG_REQUEST)

As **the Target firmware**,
I want **the dispatcher to handle the remaining content + config opcodes**,
So that **OLAF (via master) can push live category data, reconfigure rotation, and query diagnostics — completing FR10–FR14 and FR19 over the wire.**

**Acceptance Criteria:**

**Given** `target_dispatcher` extended with handlers for `0x03 REFRESH_SLOT`, `0x04 ROTATION_CONFIG`, and `0x09 DIAG_REQUEST`
**When** a REFRESH_SLOT envelope arrives with body `{slot: 3, kind: "metric_a"|"metric_b"|"chart", payload: {...}}`
**Then** the handler writes the payload into `category_store.slot(3).<kind>` and marks the slot dirty so the renderer picks it up on next frame
**And** the store accepts any `slot` between 1 and N_MAX without a slot-specific code branch (FR10)

**Given** a ROTATION_CONFIG envelope with body `{interval_ms?: 8000, order?: [1,3,2,4], active?: [1,2,4], pause?: 2|null}`
**When** received
**Then** each present field is applied to the `category_rotator` immediately; missing fields are left untouched (partial updates honoured)
**And** the change takes effect within ≤ 500 ms of receipt (NFR-P4, FR19)

**Given** a DIAG_REQUEST envelope
**When** received
**Then** the target responds with an ESP-NOW event frame (opcode `0x80+` per `espnow-wire-protocol.md`) containing firmware version, node id, uptime, free heap, RSSI, and active slot index

**Given** any REFRESH_SLOT or ROTATION_CONFIG body with an unknown optional field (e.g., `"flourish": true`)
**When** received
**Then** the dispatcher ignores the unknown field and processes the rest normally (NFR-I3 — unknown fields additive)

### Story 4.2: Last-known-good cache, staleness monitor, and three visual states

As **the Viewer**,
I want **the dashboard to keep rendering honestly when the wire goes quiet — fresh vs stale vs live**,
So that **a network blip or OLAF outage never turns into a blank display or a stale number presented as current (NFR-R3).**

**Acceptance Criteria:**

**Given** `spine/components/store_lkg/` committed — RAM-only, never persisted to flash (architectural rule)
**When** any REFRESH_SLOT or ROTATION_CONFIG payload arrives
**Then** the cache records `(slot_id, kind) → (payload, last_update_ms)` and overwrites any prior value

**Given** `components/render_lifecycle/StalenessMonitor` committed
**When** the monitor ticks (default every 5 s) and finds a slot whose `last_update_ms` is older than the configurable threshold
**Then** the target dims that slot's StatusRing colour and eventually promotes its status token to Unknown after the second threshold (FR32)
**And** thresholds are runtime-tunable via a future ROTATION_CONFIG extension (no hardcoded numbers in the renderer)

**Given** three visual states defined: `FRESH_BOOT` (no payload ever received for slot), `STALE` (payload received but older than threshold), `LIVE` (payload received within threshold)
**When** a slot transitions between states
**Then** the renderer updates ring, numerals, and timestamp atomically — no partial state is visible (centre/ring independence from FR6 still honoured)

**Given** master link loss (no envelopes for ≥ threshold)
**When** observed
**Then** every slot enters STALE and eventually Unknown; the renderer never blanks, never fabricates values, and never shows a "—" value without its grey ring + stale timestamp (FR34)

### Story 4.3: Sleep manager and deep-sleep entry

As **the Target firmware**,
I want **a clean deep-sleep entry path triggered only by a SLEEP opcode**,
So that **the node draws ≤ 0.5 mA when OLAF takes the house offline (NFR-Pw4), and never self-sleeps (FR22).**

**Acceptance Criteria:**

**Given** `spine/components/sleep_manager/` committed
**When** a `0x02 SLEEP` envelope is dispatched
**Then** the manager sequentially: disables the backlight (PWM → 0), stops the LVGL render task, flushes any pending serial log, tears down WiFi (if `AWAKE_WIFI` state), unregisters non-RTC ESP-NOW peers, arms the ESP-NOW RTC receiver, and calls `esp_deep_sleep_start()`

**Given** the target in deep sleep
**When** current is measured at the 5 V USB-C input with a multimeter (5-minute average)
**Then** draw ≤ 0.5 mA (NFR-Pw4)

**Given** the target in any awake state (`AWAKE_ESPNOW` or `AWAKE_WIFI`)
**When** no SLEEP envelope has been received
**Then** the target never initiates sleep on its own — no timer, no idle-detector, no night-mode schedule triggers sleep (FR22)

**Given** a SLEEP arrives mid-rotation or mid-transition
**When** processed
**Then** the manager aborts the in-progress transition cleanly (no torn frame left on screen) before entering sleep

### Story 4.4: Wake from deep sleep and re-render from master re-push

As **the Target firmware**,
I want **to wake cleanly on an ESP-NOW WAKE packet, re-peer, and render the first coherent frame within 3 seconds**,
So that **Journey 1 (morning glance after overnight sleep) works end-to-end (NFR-P2).**

**Acceptance Criteria:**

**Given** the target in deep sleep with the RTC ESP-NOW receiver armed
**When** a `0x01 WAKE` envelope arrives from the whitelisted master MAC
**Then** the CPU wakes, boot detects `ESP_SLEEP_WAKEUP_ESP_NOW`, and follows a wake-from-sleep path that skips normal fresh-boot (LVGL + displays re-init, tokens preloaded, last in-flash version info intact)

**Given** the wake path
**When** complete
**Then** the target sends a `0x80 WAKE_ACK` event back to the master and begins listening for re-pushed state (REFRESH_SLOT, ROTATION_CONFIG)

**Given** the master receives WAKE_ACK and flushes any queued state (Epic 4.6)
**When** the target receives the re-pushed payloads
**Then** the LKG cache is populated and the renderer transitions from FRESH_BOOT into LIVE for each received slot

**Given** the full wake cycle instrumented with timestamps (WAKE receipt at RTC → first coherent frame on all three displays)
**When** measured with a logic analyser
**Then** elapsed time ≤ 3 s (NFR-P2)

**Given** a wake packet from any MAC other than the whitelisted master
**When** received at the RTC receiver
**Then** it is dropped without waking the CPU beyond the RTC listener (FR23 enforced at RTC level, not just application layer)

### Story 4.5: On-demand WiFi lifecycle on target (WIFI_UP / WIFI_DOWN + TCP listener)

As **the Target firmware**,
I want **WiFi to be off by default when awake, and come up only when the master explicitly requests it**,
So that **the steady-state awake power profile stays near 60–70 mA and WiFi+ESP-NOW only coexist during OTA or bulk transfers.**

**Acceptance Criteria:**

**Given** `spine/components/net_wifi_lifecycle/` committed (replaces Epic 2's interim `net_wifi_persistent_target`, which is retired in this epic)
**When** the target enters `AWAKE_ESPNOW` from any path (boot or wake)
**Then** WiFi hardware is **off**; only ESP-NOW is active

**Given** a `0x06 WIFI_UP` envelope with body `{timeout_s: 30, purpose: "ota"|"bulk"}`
**When** received
**Then** the target enables WiFi, associates with NVS creds, opens a TCP listener on a known port (from `net_tcp_server`), and replies via ESP-NOW with a `0x81 WIFI_READY` event containing `{ip, port}`
**And** the target transitions to `AWAKE_WIFI` state

**Given** the target in `AWAKE_WIFI` with no TCP traffic for `timeout_s` seconds
**When** the idle timer fires
**Then** the target tears WiFi down, returns to `AWAKE_ESPNOW`, and emits a `0x82 WIFI_DOWN_EVT` event

**Given** a `0x07 WIFI_DOWN` envelope received while in `AWAKE_WIFI`
**When** processed
**Then** any in-progress TCP session is aborted cleanly and WiFi is torn down immediately

**Given** the target in `AWAKE_WIFI`
**When** current is measured at 5 V
**Then** draw is transient ~100 mA (ESP-NOW + WiFi + TCP listener active) — acceptable because `AWAKE_WIFI` is used briefly during OTA/bulk

### Story 4.6: Master transport selector, offline queue, and ESP-NOW fragmentation

As **the Master firmware**,
I want **automatic transport selection by payload size, buffering for sleeping targets, and fragmentation for 250 B–~2 KB payloads**,
So that **higher layers (MCP tools, debug commands) never worry about wire choice, and messages survive sleep gaps.**

**Acceptance Criteria:**

**Given** `nodes/node-00-master/firmware/components/transport_selector/` upgraded from its Epic 3 stub
**When** called with `(logical_name, opcode, body_size)`
**Then** it returns `TRANSPORT_ESPNOW_DIRECT` for `body_size ≤ 240`, `TRANSPORT_ESPNOW_FRAG` for `240 < body_size ≤ 2048`, and `TRANSPORT_WIFI_TCP` for `body_size > 2048` or when opcode is OTA (per architecture AR13)

**Given** `spine/components/net_espnow_frag/` committed
**When** a body >240 B is sent
**Then** the sender splits into N fragments sharing a seq_id and encodes `frag_index`, `last` bit in the envelope's seq/fr field
**And** the receiver reassembles in RAM; fragments arriving out of order are buffered; missing fragments trigger a resend request (application-layer, via `util_backoff`)

**Given** `nodes/node-00-master/firmware/components/offline_queue/` committed
**When** the operator / MCP / debug issues a send to a target whose state is `offline` (from the Epic 3 link-state table)
**Then** the envelope is queued with a TTL (default 5 min) instead of being dropped, and the caller receives a synchronous `queued` response

**Given** an offline target
**When** it ACKs a WAKE (4.4 flow) and its state transitions to `online`
**Then** the offline_queue flushes all queued envelopes for that target in arrival order, each still subject to the transport_selector

**Given** queued envelopes exceeding TTL
**When** the queue tick runs
**Then** expired entries are dropped and a single serial log line records the drop

### Story 4.7: Master-streamed OTA (supersedes Epic 2's direct pull)

As **Kamal (Operator)**,
I want **OTA firmware delivered from master → target over WiFi TCP on demand, triggered by an OTA_BEGIN opcode**,
So that **the target has no direct URL or HTTP pull path, and OTA flows through the same master-routed control plane as everything else.**

**Acceptance Criteria:**

**Given** `nodes/node-00-master/firmware/components/ota_streamer/` committed
**When** the master is asked to OTA `dashboard_a` with a firmware binary and expected SHA-256
**Then** the streamer: (a) sends `0x06 WIFI_UP` with `purpose="ota"`, (b) awaits `0x81 WIFI_READY` with `{ip, port}`, (c) opens a TCP connection, (d) sends `0x08 OTA_BEGIN` via ESP-NOW with `{sha256, size}`, (e) streams the binary over the TCP socket, (f) closes TCP, (g) sends `0x07 WIFI_DOWN`

**Given** `ota_manager` on the target (retained from Epic 2.3 but re-pointed)
**When** OTA_BEGIN arrives followed by TCP chunks
**Then** the manager accumulates bytes into the inactive partition, verifies SHA-256 on completion, activates on match, reboots, and self-rolls-back on failed boot (same mechanism as 2.3, different source)

**Given** Epic 2's direct-pull OTA path
**When** Epic 4 lands
**Then** the direct-pull code path is removed from `ota_manager` (not `#ifdef`'d — deleted) and any direct-pull URL logic in NVS provisioning is removed from `factory/credentials.env.example`

**Given** a mid-stream failure (master WiFi drops, TCP disconnect, power blip)
**When** the target is observed
**Then** it never activates a partial partition; the running firmware continues; a single honest error line is logged and the updating visual state exits back to normal rendering

**Given** OTA across a 1.5 MB binary
**When** timed end-to-end
**Then** ≤ 30 s from OTA_BEGIN to reboot (practical target — architecture states ~10 s over WiFi; budget includes WIFI_UP handshake)

### Story 4.8: Diagnostic event stream (target → master)

As **the Master firmware**,
I want **a structured event stream from each target carrying logs, state transitions, and keepalives**,
So that **Epic 5's MCP `node_logs` resource and `fleet_status` have live data to expose without changing the wire format later.**

**Acceptance Criteria:**

**Given** `spine/components/diag_manager/` committed on the target
**When** any module emits a log via `LOG_SAFE()` at configured verbosity or higher
**Then** `diag_manager` wraps the line into a `0x80+` event envelope (with level, timestamp, message) and sends via ESP-NOW to the master

**Given** target state transitions (FRESH_BOOT → LIVE per slot, AWAKE_ESPNOW ↔ AWAKE_WIFI, entering sleep)
**When** each occurs
**Then** an event is emitted to the master with the new state

**Given** a keepalive tick on the target (default every 30 s while `AWAKE_ESPNOW`)
**When** it fires
**Then** a `0x83 KEEPALIVE` event is sent containing RSSI, free heap, uptime, active slot

**Given** master-side event handling
**When** events arrive
**Then** the master updates routing-table `last_seen_ms` and `state`, and forwards the raw event structure to any connected consumer (serial log for now; WebSocket forward added in Epic 5.1)

**Given** secret-tagged values in any emitted log
**When** transmitted
**Then** they are redacted by `LOG_SAFE()` before the envelope is packed — never on the wire (NFR-S6)

### Story 4.9: Auto-reconnect, backoff, and 72-hour soak

As **Kamal (Operator)**,
I want **the target (and master) to survive routine disruptions without operator intervention and prove it over a 72-hour run**,
So that **the MVP exit gate (72 h wall soak) is reached with measured data, not optimism.**

**Acceptance Criteria:**

**Given** `spine/components/util_backoff/` committed
**When** any network operation fails (WiFi associate, ESP-NOW peer register, TCP connect)
**Then** the caller retries with exponential backoff (starting 1 s, capping at 60 s, ±20% jitter)

**Given** the target in `AWAKE_WIFI` and the AP drops
**When** observed
**Then** WiFi reconnects within ≤ 60 s of AP recovery without operator intervention (NFR-R2, FR26)

**Given** the master and a target both running
**When** the master is power-cycled
**Then** the target's ESP-NOW peer remains healthy; once master returns, normal traffic resumes without the target needing a reboot

**Given** a 72-hour continuous run with the target mounted and driven by master dummy-content script
**When** the run completes
**Then** uptime ≥ 99% (NFR-R1), heap free never drops below a committed floor over the run (NFR-R4), no crash loops, no memory leaks, and every Epic 4 opcode has been exercised at least once during the run

**Given** deliberately malformed envelopes (truncated body, bad magic, impossible opcode) injected during the soak
**When** received
**Then** the target logs and discards each without crash; LKG is retained (NFR-R6)

### Story 4.10: Performance gate — NFR-P1 through NFR-P6 measured

As **Kamal**,
I want **every performance NFR from the PRD measured with a committed instrumentation method and numbers recorded**,
So that **Milestone 3 is not "done when it feels ready" but "done when the gates pass."**

**Acceptance Criteria:**

**Given** a tripod-mounted 240 fps video of a rotation event
**When** reviewed frame-by-frame
**Then** all three displays begin their cross-fade within ≤ 50 ms of each other (NFR-P3)

**Given** a logic-analyser trace of a WAKE packet received → first coherent frame on all three displays
**When** measured across 10 runs
**Then** every run is ≤ 3 s; median is recorded (NFR-P2)

**Given** a timestamped log of ROTATION_CONFIG envelope receipt and the matching rotation behaviour change
**When** measured across 10 trials (vary interval, order, pause, active)
**Then** every trial shows effect within ≤ 500 ms of receipt (NFR-P4)

**Given** a single-display redraw (full frame — chart + ring + numerals) instrumented at the LVGL flush boundary
**When** measured per chart type (line / scatter / bar)
**Then** every variant completes in ≤ 100 ms on the ESP32-S3 at 40 MHz SPI clock (NFR-P5)

**Given** a cold USB-C power-on event (target unprovisioned-free boot)
**When** measured from power-applied to first honest placeholder frame on all three displays
**Then** ≤ 5 s (NFR-P6)

**Given** Kamal and one family member per category at 3 m from the mounted dashboard
**When** each reads the primary metric value and status for each category
**Then** time-to-read ≤ 2 s, verified across at least 8 observations (NFR-P1)

**Given** all measurements taken
**When** the soak + perf run is complete
**Then** numbers are recorded in `documents/implementation-artifacts/epic-4-perf-gate.md` with dates, methodology, and raw data references

## Epic 5: MCP Agent Surface

Expose the fleet to MCP-capable agents (OLAF, Claude, scripts). Python MCP server speaks HTTP/SSE to agents and WebSocket to the master; master's `ws_mcp_endpoint` listens on `:8080`; six tools address `dashboard_a` by logical name; `fleet_status` and `node_logs` resources surface live state. Exit gate: Journey 4 passes end-to-end — a brand-new category (e.g., "Sleep" as slot 5) renders on firmware that has never heard of it.

### Story 5.1: Master WebSocket server (`ws_mcp_endpoint`) on :8080

As **the Master firmware**,
I want **a WebSocket server exposing the MCP command and event frames**,
So that **the Python MCP server has a single transport for commands out and events in.**

**Acceptance Criteria:**

**Given** `spine/components/net_ws_server/` committed (used by master only) and `nodes/node-00-master/firmware/components/ws_mcp_endpoint/` wiring it up
**When** the master boots and persistent WiFi associates
**Then** a WebSocket server listens on port 8080 (configurable via NVS `mcp/ws_port`) and accepts any number of clients on the LAN (no auth for MVP, per architecture AR17)

**Given** an incoming command frame `{"id":"req-42","op":"send","target":"dashboard_a","action":"refresh_slot","payload":{...}}`
**When** received
**Then** the endpoint resolves `dashboard_a` via the routing table, dispatches via `transport_selector`, and replies with `{"id":"req-42","ok":true,"transport":"espnow","latency_ms":<measured>}` — the request/response frame shapes match `docs/master-ws-protocol.md`

**Given** any target-to-master diag event from Epic 4.8
**When** received by the master
**Then** it is forwarded to every connected WS client as a `{"event":"...","target":"...","...":"..."}` frame
**And** clients that disconnect mid-flow do not block forwarding to other clients or crash the master

**Given** `docs/master-ws-protocol.md` committed
**When** reviewed
**Then** it documents every supported `op` value, every event shape, error codes, and reconnection expectations — this doc is the contract, not the code

### Story 5.2: Python MCP server scaffold and WebSocket client to master

As **Kamal (Operator)**,
I want **a Python MCP server package with a committed WebSocket client to the master**,
So that **MCP tools (5.3) have a connection to build on and the MCP package has a buildable starting point.**

**Acceptance Criteria:**

**Given** `mcp/` subtree committed with `pyproject.toml` (uv or poetry), `olaf_mcp/` package, and `README.md`
**When** `uv sync` (or `poetry install`) runs
**Then** all dependencies resolve and `python -m olaf_mcp.server` starts without error

**Given** `olaf_mcp/server.py` exposing an MCP HTTP/SSE endpoint per Anthropic's MCP spec
**When** an MCP-capable client connects
**Then** the server completes the MCP handshake and lists zero tools/resources (tools land in 5.3, resources in 5.4)

**Given** `olaf_mcp/master_client.py` — a WebSocket client
**When** `server.py` boots
**Then** `master_client` connects to `ws://<master-ip>:8080`, reconnects with exponential backoff on drop, and exposes an async `send(op, target, action, payload) → response` API and an event subscription callback

**Given** `olaf_mcp/config.py`
**When** loaded
**Then** it reads `MASTER_WS_URL`, `MCP_HOST`, `MCP_PORT`, and log level from environment variables; a committed `mcp/config.env.example` documents each variable

### Story 5.3: Six tools for `dashboard_a` with logical-name routing

As **OLAF (or any MCP-capable agent)**,
I want **six MCP tools to drive `dashboard_a` by logical name**,
So that **I can refresh content, reconfigure rotation, adjust brightness, wake, sleep, and OTA without ever touching a MAC.**

**Acceptance Criteria:**

**Given** `olaf_mcp/tools/dashboard_a.py` committed with tools `refresh_slot`, `set_rotation`, `set_brightness`, `wake`, `sleep`, `ota`
**When** any tool is invoked with a valid payload
**Then** it maps to the corresponding `op` on the master WebSocket (per `docs/mcp-tool-reference.md`), awaits the response, and returns a structured result to the agent

**Given** `olaf_mcp/tools/__init__.py` with an auto-discovery registrar
**When** a new `tools/<node>.py` file is added (e.g., future `dial_b.py`)
**Then** its exported tools register with the MCP server on next restart without edits to `__init__.py`

**Given** any tool invocation
**When** it completes
**Then** the tool NEVER exposes MAC addresses, encryption keys, or wire-level opcode numbers in its input schema, output, or error messages (agents see logical names only — architecture AR19)

**Given** `refresh_slot` called with a slot that exceeds the target's configured N_MAX
**When** processed
**Then** the tool returns a structured error with a human-readable message and does not crash or time out

**Given** `docs/mcp-tool-reference.md` committed
**When** reviewed
**Then** every tool lists its JSON schema input, expected output, failure modes, and a minimal usage example

### Story 5.4: MCP resources — `fleet_status` and `node_logs`

As **OLAF**,
I want **read-only resources exposing fleet state and live logs**,
So that **I can make routing decisions (e.g., don't push to an offline target) and surface node diagnostics to Kamal without writing custom tool calls.**

**Acceptance Criteria:**

**Given** `olaf_mcp/resources/fleet_status.py` committed
**When** the resource is read
**Then** it returns a JSON document listing every target in the routing table with `logical_name`, `state` (online/offline/sleeping), `last_seen_ms`, `rssi` (from most recent keepalive), `fw_version`, `active_slot`
**And** it never includes MACs or keys

**Given** `olaf_mcp/resources/node_logs.py` committed
**When** subscribed via MCP
**Then** it streams diag events forwarded by the master's WebSocket server (Epic 4.8 + 5.1) filtered by target, with configurable level floor (INFO/WARN/ERROR)

**Given** an agent subscribing to `node_logs` for `dashboard_a`
**When** a WARN-level LOG_SAFE line is emitted from the target
**Then** the agent receives it within ≤ 2 s of the target emitting it (the latency budget is MCP → master → target round-trip, dominated by the MCP HTTP/SSE layer)

### Story 5.5: systemd service and deployment

As **Kamal (Operator)**,
I want **a committed systemd unit that runs the MCP server as a service**,
So that **the MCP layer is always available without manual starts and survives reboots of the MCP host.**

**Acceptance Criteria:**

**Given** `mcp/systemd/olaf-mcp.service` committed
**When** installed via the committed `mcp/README.md` instructions (`cp` + `systemctl enable --now olaf-mcp`)
**Then** the service starts, reads environment from `/etc/olaf-mcp/config.env` (or equivalent), and survives a reboot of the host

**Given** the service running
**When** the master is unreachable at boot
**Then** the MCP server still starts successfully, logs the connection failure, and begins backoff — agents connecting receive tool listings but calls return transport-unavailable errors until the master is reachable

**Given** the committed `mcp/README.md`
**When** followed on a fresh MCP host (Pi or laptop)
**Then** the operator can go from clone → service running in < 10 minutes with only `MASTER_WS_URL` configuration

### Story 5.6: Journey 4 end-to-end validation — platform thesis exit gate

As **Kamal**,
I want **a recorded demonstration that OLAF can push a brand-new category to the dashboard without any firmware rebuild**,
So that **the platform thesis (dumb renderer + smart brain + AI-directed hardware as a durable contract) is validated, not asserted.**

**Acceptance Criteria:**

**Given** a production-flashed target running Epic 4 firmware (compiled before "Sleep" existed as a category name anywhere in firmware source — verified by grep)
**When** an agent (Claude or a test script via MCP) invokes `refresh_slot(target="dashboard_a", slot=5, kind="metric_a", payload={"value":"7h 12m","unit":"","label":"Sleep","status":"good","timestamp_text":"last night"})` followed by `metric_b` and `chart` with `type="bar"`, then `set_rotation(active=[1,2,3,4,5])`
**Then** on the next rotation the dashboard renders category 5 with both rounds and a 14-night bar chart, no restart, no reflash, no wire-level intervention

**Given** the same end-to-end test exercised for each of the other five MCP tools (`set_brightness`, `set_rotation` alone, `wake`, `sleep`, `ota`)
**When** each tool is invoked against `dashboard_a`
**Then** the visible behaviour matches the PRD's described outcome for that capability

**Given** an `ota` tool invocation delivering a fresh build
**When** completed
**Then** the post-OTA fw_version reported in `fleet_status` is the new build's version, and the target is rendering the same LKG it had before OTA (via master re-push after reboot)

**Given** `documents/implementation-artifacts/epic-5-journey4-gate.md` committed
**When** reviewed
**Then** it contains a dated walk-through with screenshots / photos of each step, the exact MCP tool calls used, the target's fw_version at the time of the test, and a grep-verified claim that no "Sleep" string existed in firmware source when the build was produced


