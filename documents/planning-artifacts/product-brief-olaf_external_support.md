---
title: "Product Brief: OLAF External Support"
status: "complete"
created: "2026-04-18"
updated: "2026-04-18"
inputs:
  - "Conversation with Kamal (2026-04-18)"
  - "OLAF Health Dashboard Product Brief v1.1"
  - "OLAF GitHub repository (github.com/kamalkantsingh10/OLAF)"
---

# Product Brief: OLAF External Support

## Executive Summary

OLAF is a personality-first companion robot — a self-balancing, SLAM-navigating, Pi 5 + Hailo + ROS2 robot with expressive eyes, animated displays, and a voice. OLAF is the collaborator; OLAF is the brain. But OLAF's body is small, and he can only be in one room at a time.

**OLAF External Support is the family of custom ESP32-based hardware nodes that OLAF commands to set up the room** — making physical spaces around the home glanceable, sensed, and controllable, even when OLAF himself is three rooms away. The platform's first (and currently only) node is the **Health Dashboard**, a three-screen rotating ambient display for personal health metrics. Additional archetypes (sensor, controller, media) are possible future directions, not current commitments.

The platform's defining principle: **every node is an instrument OLAF plays.** Firmware does not hardcode what to show or what to do — it exposes a vocabulary of rendering primitives that OLAF composes at runtime via MQTT. OLAF decides what data matters and how it appears; the node simply renders. The framework is designed to give OLAF maximum leeway to push content, visualizations, and states over MQTT without firmware changes.

## Current State & Scope

OLAF himself is still under active development. This brief describes the External Support platform *as it will be* once OLAF's MQTT publisher and data pipeline are in place — but the Health Dashboard is designed to be **useful on day one, even with a partial OLAF.** See *Offline & Degraded Behaviour* below.

**In scope right now:** the Health Dashboard node — hardware, firmware, and the MQTT framework that connects it to OLAF.

**Not in scope right now:** voice / conversational interaction (OLAF does not speak to, or freeze, the dashboard via voice in this project); other node archetypes; multi-room replication; data ingestion pipeline design.

## The Problem

OLAF is a genuinely capable embodied AI. He can hold a conversation, express mood, navigate rooms, and reach cloud LLMs for reasoning. But three practical limits keep him from being as useful as he could be:

1. **His body is a small canvas.** A 2-inch heart display and two animated eyes are wonderful for personality — useless for a 30-day weight trend or a 14-day glucose scatter plot.
2. **He can only be in one place.** If OLAF is charging in the study, a kitchen node is the only way anyone in the kitchen sees what OLAF knows.
3. **Voice is an expensive interaction for passive glances.** Asking "OLAF, what's my glucose?" every time is friction. An ambient dashboard on the wall costs nothing to read.

Off-the-shelf alternatives (Home Assistant dashboards, smartwatches, Apple Health widgets) each solve fragments — but none are **OLAF-native**. They cannot be re-directed by OLAF on a whim, cannot participate in OLAF's expression layer, and do not share OLAF's architectural spine.

## The Solution

OLAF External Support is a family of stationary, single-purpose hardware nodes — each one cheap to build, aesthetically enclosure'd, and designed from day one to be **AI-directed**. Every node speaks the same MQTT conventions to OLAF, wakes over ESP-NOW (so OLAF can reach it even when WiFi is flaky), caches last-known-good state, and runs "dumb renderer" firmware: OLAF owns the meaning, the node owns the execution.

The Health Dashboard — the first node — lives on the wall and shows Kamal's personal health metrics in a rotating ambient display. When OLAF has new data to share (glucose reading, weight update, fasting progress), he publishes it on MQTT and the dashboard incorporates it instantly. When OLAF wants to highlight a different metric, change the chart type, or dim the display at night, he publishes that directive — and the dashboard obeys. The household becomes OLAF's extended body.

## What Makes This Different

| Principle | What it means in practice |
|---|---|
| **AI-directed hardware** | Firmware ships with a vocabulary of rendering and control primitives. OLAF composes that vocabulary at runtime via MQTT. Switching a screen from line chart to bar chart, or triggering a pre-defined animation, is a message — not a firmware release. *Genuinely new primitives (e.g., a liquid-fill animation that doesn't exist yet) still require a firmware OTA — but adding one unlocks it for every future OLAF directive.* |
| **Dumb renderer, smart brain** | Zero domain logic in firmware. Nodes hold no knowledge of what "glucose" or "fasting" mean. OLAF pushes content, status, and thresholds; the node obeys. |
| **One spine, many forms** | Every node uses ESP32 + MQTT (data) + ESP-NOW (wake) + last-known-good caching + PWM brightness + deep sleep. A new node type inherits the spine; only the form factor and rendering primitives change. |
| **Family-usable** | Nodes must work for non-builders in the household. Glance-readable, ritual-free, no phone required. |
| **Teachable longevity** | A 2-year-old node should still be directable by OLAF in ways Kamal didn't anticipate — within the primitive vocabulary shipped in firmware. Vocabulary expansion is a deliberate OTA event, not a workaround. |
| **Ambient, not conversational** | Principled rejection of "ask the AI every time." For data OLAF already knows, glanceable surfaces beat conversation. The dashboard is pushed-to, not asked. |

## Who This Serves

**Primary:** Kamal — as builder, as daily user, and as the person who will add nodes over the coming months.

**Secondary (and growing):** Kamal's family — real end users. As OLAF matures, the nodes are how non-builders in the household benefit from OLAF without having to talk to him.

**Not in scope:** Commercial customers, open-source community builders, or a product for sale. This is a personal platform. Documentation may become public later, but that is not a design driver.

## Node Archetypes — the Platform Shape

The platform is designed around four archetypes. **Only Info Surface (Health Dashboard) is being built in this project.** The others are named here to document the architectural thinking, not as commitments.

| Archetype | Role | Status |
|---|---|---|
| **Info surface** | Renders data OLAF pushes | **In scope — Health Dashboard (Node 1)** |
| **Sensor** | Feeds ambient data into OLAF | Future possibility, not committed |
| **Controller** | OLAF actuates other devices through it | Future possibility, not committed |
| **Media** | OLAF's audio presence in a room | Future possibility, not committed |

## First Node: Health Dashboard

The Health Dashboard is the platform's flagship and architectural reference. Its detailed spec (previously circulated as *v1.1*) is folded in here by essence.

- **Form factor:** Three-screen ESP32 node — two 2.28" round TFTs (GC9A01, 240×240) for metric summaries, one 2.8" rectangular TFT (ILI9341, 320×240) for charts. USB-C powered, always-on, plastic/acrylic enclosure.
- **Experience:** At any moment, all three screens display one health category together — Round A shows a headline metric, Round B a secondary metric, the rectangle shows a chart. Every ~10 seconds (configurable by OLAF), all three screens transition together to the next category. Viewer always sees a coherent snapshot of one domain at a time.
- **Reference categories:** Body Weight · Glucose · Cycling · Fasting. Definitions, units, chart types, and thresholds are all pushed by OLAF — zero firmware changes to add, rename, or reorder categories.
- **Status ring:** Each round display renders a coloured outer ring (green / amber / red / grey) reflecting whether the current value is in a healthy range. OLAF evaluates thresholds and sends the resolved status — node simply maps status → colour.
- **Interaction model:** Pure push. OLAF publishes data and directives; the dashboard obeys. No voice commands, no user input on the node itself — the dashboard is read, not touched.
- **Framework flexibility:** MQTT topic schema is designed to give OLAF maximum leeway — category definitions, metric values, chart types, status thresholds, rotation timing, brightness, and pause state are all pushed from OLAF. No node-side logic about *what* health means.
- **Power model:** Backlight 0% (≈100 mA, WiFi connected) is the primary idle state. Deep sleep (≈0.5 mA) is for overnight / OLAF-off. ESP-NOW wake is the only way out of deep sleep.

A detailed technical specification — MQTT topic schema, SPI wiring, state machine, rendering layout — lives in a separate engineering document and is not repeated here.

## What Happens Next

**Already in hand:** A 3D-printed enclosure for the three-display arrangement.

**Immediate (days):** Hardware build — ESP32, two GC9A01 rounds, one ILI9341 rectangle, SPI fan-out wired up inside the enclosure.

**Following (own pace, no timeline):**
- **Visual / on-screen design** — typography, layout, colour palette, status ring styling, transition animations, chart aesthetics. Each category needs a look that is calm enough to live on a wall and clear enough to be glanced at.
- **Firmware** — built around the MQTT framework that lets OLAF direct everything. Implements the primitive vocabulary (chart types, status colours, rotation, brightness) and the offline/degraded behaviours.
- **OLAF-side publisher** — develops in parallel as OLAF itself matures.

**Later (possibility space, not commitment):** Additional archetypes or dashboard variants come into scope only when the Health Dashboard is stable, the framework is proven, and a real need emerges. The architecture leaves the door open; the roadmap does not walk through it yet.

## Success Criteria

### Health Dashboard — concrete & testable

1. Three displays rotate coherently through 4 health categories at configurable cadence (default 10s).
2. On ESP-NOW wake from deep sleep: node is rendering last-known-good within 2 seconds.
3. Survives a 24-hour WiFi or OLAF outage displaying last-known-good data with a clearly communicated staleness indicator.
4. Category definitions, metrics, thresholds, chart types, rotation order, and active-category set can be changed entirely over MQTT — no firmware flash required.
5. OLAF can pause rotation on a specific category and resume it via MQTT directive.

### Platform framework — aspirational & directional

1. **The MQTT framework gives OLAF maximum leeway.** OLAF can direct the dashboard within its primitive vocabulary without firmware changes. Vocabulary *expansion* (genuinely new primitives like novel animations) is a deliberate OTA event.
2. **Family uses the dashboard without asking how.** A node that needs explaining is a failure.
3. **OLAF feels present even when he's in another room.** The household behaves as though OLAF is everywhere — because the dashboard makes that true.

## Offline & Degraded Behaviour

OLAF is under active development and will reboot, stall, or be unreachable more often than a mature system. The broker, WiFi, and OLAF are three separate points of possible failure. Nodes must remain trustworthy — or calibrated honestly — under each:

- **OLAF unreachable, broker up:** Keep rendering last-known-good. Show an unobtrusive staleness indicator (e.g., dim the status ring after N minutes without update).
- **Broker down, WiFi up:** Same as above — last-known-good is the node's only truth.
- **WiFi down:** Same again. ESP-NOW channel remains listening for wake/reset from OLAF when the network recovers.
- **Fresh boot, no prior data:** Grey status rings, explicit "waiting for OLAF" placeholder rather than blank or misleading numbers.

Every node has a standalone mode that degrades gracefully. A node must never be more wrong than honestly stale.

## Open Questions

Live design tensions worth resolving during Health Dashboard firmware work:

- **Firmware vocabulary shape — semantic, compositional, or hybrid?** Semantic: OLAF sends `chart_type: "bar"` from a fixed catalog. Compositional: OLAF sends drawing primitives (draw_arc, fill_to, fade) and sequences them. Hybrid: both. **Current leaning:** semantic for the first firmware, with the MQTT schema designed so compositional primitives can be added later without a breaking change.
- **Data ingestion into OLAF** — deliberately deferred in this project. The Health Dashboard's MQTT framework is designed to be neutral on where the data originally came from; OLAF can use a public library if one exists, or ingest data by any path he chooses. The dashboard does not care.
- **Privacy surface** — a wall dashboard showing glucose or weight is visible to guests and visitors. Options: a privacy mode toggle, design metrics to be ambiguous to outsiders (status ring without number), or OLAF-controlled discretion directives.
- **OTA firmware channel** — assumed but unspecified. Worth designing in early so vocabulary expansion remains cheap.

## Deliberately Deferred

To keep ambition preserved without being promised:

- Compositional / generative rendering primitives beyond a fixed catalog.
- Voice and conversational interaction with the dashboard (OLAF pushes; the dashboard does not listen).
- Sensor, controller, and media archetypes — architecturally planned for, not built.
- Multi-room dashboard replication and per-node overrides.
- Choreographed cross-node experiences.
- Open-source documentation or public product packaging.
