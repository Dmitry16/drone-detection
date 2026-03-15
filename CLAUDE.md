# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **drone/airborne object detection system** for ground robotic platforms. The goal is to detect approaching airborne objects (drones, aircraft, birds) as early as possible and convert detections into actionable situational awareness. The project is currently in architecture/documentation phase with no implementation code yet.

## Repository Structure

```
drone-detect/
├── README.md
└── docs/
    └── airborne_detection_pipeline.md   # Full 7-stage pipeline design (401 lines)
```

## Architecture

The system follows a **7-stage pipeline**:

| Stage | Name | Purpose |
|-------|------|---------|
| 0 | Sensor Ingest | Multi-camera RGB capture, IMU sync, hardware decode |
| 1 | Video Conditioning | IMU stabilization, CLAHE, ROI masking |
| 2 | Candidate Generation | Frame differencing, optical flow, blob extraction |
| 3 | AI Detection | Classifier on crops or tiled-frame detector |
| 4 | Multi-Object Tracking | Track lifecycle: tentative → confirmed → coasting → terminated |
| 5 | Temporal Reasoning | Approach detection, time-to-closest-approach estimation |
| 6 | Threat Scoring | Hysteresis-based alert logic, multi-target context |
| 7 | Output/Interface | Events, overlays, ring buffer, autonomy API |

**Key design decisions:**
- Run fast candidate generation (Stage 2) before AI inference — avoids full-frame detection
- Prioritize **high recall** over per-frame precision; use tracking to reduce false alarms
- Combine multiple cues (motion, AI, temporal) rather than relying on a single model
- Multi-camera coverage; IMU attitude used for sky/horizon segmentation and stabilization

**Detection classes:** `drone_like`, `bird`, `manned_aircraft`, `unknown_airborne`, `background_false_alarm`

**Output events:** `possible_air_object`, `confirmed_air_object`, `incoming_object`, `multi_object_alert`

## Implementation Notes (when building)

- No build system, test framework, or language has been selected yet — these decisions need to be made before implementing
- The pipeline design targets embedded/robotic compute, so latency and resource usage are primary constraints
- Configuration will need to cover: camera parameters, model weights, detection thresholds, alert thresholds, IMU calibration
- The full architecture spec is in `docs/airborne_detection_pipeline.md` — read it before implementing any stage
