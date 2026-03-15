
# Early Detection of Airborne Objects for Ground Robotic Platforms

Task description, Part 1 suggested solution, and Part 2 production-style computer vision / AI pipeline.

## Table of Contents
- [Problem Description](#task-description)
- [Solution 1 — System Layers High Level Description](docs/airborne_detection_pipeline.md#part-1-—-system-solution-high-level)
- [Solution 1 — Computer Vision / AI Pipeline for Early Airborne Detection (High Level)](docs/airborne_detection_pipeline.md#solution-1-cv-ai-pipeline)
  - [1. Pipeline Goal](docs/airborne_detection_pipeline.md#pipeline-goal)
  - [2. Overall Pipeline Architecture](docs/airborne_detection_pipeline.md#overall-pipeline-architecture)
  - [Stage 0 — Sensor Ingest](docs/airborne_detection_pipeline.md#stage-0-sensor-ingest)
  - [Stage 1 — Video Conditioning](docs/airborne_detection_pipeline.md#stage-1-video-conditioning)
  - [Stage 2 — Fast Candidate Generation](docs/airborne_detection_pipeline.md#stage-2-fast-candidate-generation)
  - [Stage 3 — AI Detection / Classification](docs/airborne_detection_pipeline.md#stage-3-ai-detection--classification)
  - [Stage 4 — Multi-object Tracking](docs/airborne_detection_pipeline.md#stage-4-multi-object-tracking)
  - [Stage 5 — Temporal Reasoning](docs/airborne_detection_pipeline.md#stage-5-temporal-reasoning)
  - [Stage 6 — Threat Scoring](docs/airborne_detection_pipeline.md#stage-6-threat-scoring)
  - [Stage 7 — Output and Interface](docs/airborne_detection_pipeline.md#stage-7-output-and-interface)
- [Orange Pi 5 Deep Research Report](docs/orange-pi5-deep-research-report.md#early-detection-of-airborne-objects-on-embedded-compute-with-orange-pi-5)
- [Orange Pi 4 Variant Report. Costs, technical comparison with Pi 5](docs/orange_pi4_variant_report.md#early-airborne-detection-on-embedded-compute)

---

# Problem Description

- The objective is to design a robust onboard perception system for a ground robotic platform that can detect approaching airborne objects as early as possible and convert those detections into usable situational awareness.
- The practical challenge is not only object detection, but early detection of very small targets under real outdoor conditions: clouds, trees, rooftops, glare, dusk, haze, motion blur, and platform movement all degrade performance.
- The system must therefore combine sensors, video conditioning, AI-based recognition, temporal tracking, and threat evaluation.
- It should distinguish likely airborne threats from birds, insects, visual artifacts, and background noise while operating with low latency on embedded compute and tracking multiple objects simultaneously.
- Desired outputs: stable object tracks, confidence estimates, approach indicators, threat scores, and alert events usable by an operator interface or autonomy stack.
