# Node 01 — Health Dashboard: Faceplate 3D Model

**Onshape document (public read):** https://cad.onshape.com/documents/72e6d2befaaa54f7e2857f22/w/5eca2ecbb4ddeace8d86dea3/e/5f78b0d6b05229dab1916dd4?renderMode=0&uiState=69e947eb8d84fa6dd540efa7

## Hardware envelope

- 2× GC9A01 2.28" round TFT (240×240)
- 1× ILI9341 2.8" TFT (240×320), operated in portrait
- ESP32-S3-WROOM-1 N16R8 DevKit
- Plastic / acrylic only — no metal near the PCB antenna (RF-through-plastic)

## Faceplate layout (per UX spec)

- **Left:** ILI9341 in portrait orientation (chart canvas)
- **Right:** two GC9A01 rounds stacked vertically, aligned along the faceplate's horizontal midline with the rectangle's centre

## Print settings

*To be documented in the Onshape document description.*

## Exports

STL and STEP files to be exported from the Onshape document and committed alongside this README when ready.

## Story reference

Deliverable for Epic 1 Story 1.1 — see `documents/planning-artifacts/epics.md`.
