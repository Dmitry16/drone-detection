# Part 1 — System Solution (High Level)

The recommended solution is a **layered perception architecture** combining sensors, computer vision, and temporal reasoning.

Instead of relying on a single AI model, the system uses several processing stages that gradually convert raw video data into actionable situational awareness.

## Sensor Layer
- Multi‑camera RGB coverage for panoramic observation
- Optional zoom / narrow field‑of‑view camera for long‑range confirmation
- IMU providing yaw / pitch / roll data for stabilization
- Optional thermal or low‑light sensors for dusk or night operation

## Perception Layer
- Video stabilization
- Denoise
- Exposure and contrast conditioning
- Lightweight candidate detection

## AI Layer
Produces class probabilities for:
- drone‑like object
- bird
- aircraft
- clutter
- unknown airborne object

## Tracking Layer
- Persistent track IDs
- Trajectory history
- Short‑gap tolerance
- Confidence smoothing

## Decision Layer
- Approach detection
- Threat scoring
- Hysteresis‑based alert logic
- Event generation

The perception layer stabilizes video and normalizes contrast to preserve tiny targets.  
Instead of heavy full‑frame detection, the system first searches for **candidate airborne objects using motion cues**.

The AI model then estimates probabilities for drone‑like objects, birds, aircraft, or background artifacts.  
Precision improves later using tracking and temporal logic.

Multi‑object tracking assigns each object a persistent identity, motion history, and evolving confidence.

Approach detection determines whether the object moves toward the platform using indicators such as:

- increasing apparent size
- stable bearing
- trajectory convergence

Threat scoring combines:
- classification confidence
- track stability
- motion pattern
- closing behavior
- number of simultaneous objects

Possible system events:

- suspicious airborne object
- confirmed airborne object
- incoming object
- multi‑object alert

Operational outputs include alerting, telemetry snapshots, short incident recordings, visualization overlays, and APIs for the autonomy stack.

---

<a name="solution-1-cv-ai-pipeline"></a>
# Part 2 — Computer Vision / AI Pipeline for Early Airborne Detection (High Level)

This describes a **production‑style CV stack**, not just “one model on video”.

Focus:
- very small targets
- low latency
- multiple simultaneous objects
- robustness to false positives

---

<a name="pipeline-goal"></a>
# 1. Pipeline Goal

## Inputs
- one or multiple camera video streams
- platform telemetry: IMU, yaw/pitch/roll, velocity
- frame timestamps

## Outputs
List of airborne tracks:

For each track:

- class
- confidence
- bearing / sector
- approaching_score
- threat_score
- time_to_closest_approach (or approximation)

Events:

- possible_air_object
- confirmed_air_object
- incoming_object
- multi_object_alert

---

<a name="overall-pipeline-architecture"></a>
# 2. Overall Pipeline Architecture

Recommended stages:

0. Sensor ingest
1. Video conditioning
2. Fast candidate generation
3. AI detection / classification
4. Multi‑object tracking
5. Temporal reasoning
6. Threat scoring
7. Output and control interface

---

<a name="stage-0-sensor-ingest"></a>
# Stage 0 — Sensor Ingest

- RTSP / CSI / USB camera capture
- hardware video decode
- strict timestamp synchronization
- frame association with IMU state

Important requirements:

- fixed FPS
- monotonic timestamps
- zero‑copy frames if possible
- bounded queues to avoid latency buildup

Workers:

- capture_worker
- preprocess_worker
- candidate_worker
- inference_worker
- tracker_worker
- threat_worker

Metrics:

- input FPS
- decode latency
- dropped frames
- queue depth
- end‑to‑end frame age

---

<a name="stage-1-video-conditioning"></a>
# Stage 1 — Video Conditioning

Critical stage to reduce noise before AI.

### Stabilization
- IMU‑assisted stabilization
- global motion estimation
- feature‑based frame registration

Output:
- stabilized frame
- global motion vector
- camera attitude

### Denoise + contrast normalization
- light denoise
- temporal denoise with small window
- CLAHE or local normalization
- gamma correction

### ROI masking
Segment:
- sky
- horizon
- ground textures
- building / tree edges

Goal: reduce false positives without overly aggressive masking.

---

<a name="stage-2-fast-candidate-generation"></a>
# Stage 2 — Fast Candidate Generation

Purpose: cheaply narrow the search area.

Methods:

- frame differencing
- motion saliency
- optical flow
- background residual
- tiny blob extraction

Process:

1. compare stabilized frame t and t‑1
2. compensate global motion
3. compute residual
4. filter noise
5. keep small connected components
6. generate candidate ROIs

Good candidate characteristics:

- small size
- motion relative to stabilized background
- persistent for 2–3 frames
- not matching static artifacts
- not sensor noise

Important rule:

**Do not run a heavy detector on every full frame.**

---

<a name="stage-3-ai-detection--classification"></a>
# Stage 3 — AI Detection / Classification

Two options:

### A — Detector on tiled frame
Pros:
- better tiny‑object detection

Cons:
- heavier compute
- higher latency

### B — Classifier or micro‑detector on candidate crops
Pros:
- cheaper
- faster
- scalable

Cons:
- depends on candidate quality

Recommended MVP:

candidate generator + crop classifier.

Minimum classes:

- drone_like
- bird
- manned_aircraft
- unknown_airborne
- background_false_alarm

For early detection, **high recall is more important than per‑frame precision**.

---

<a name="stage-4-multi-object-tracking"></a>
# Stage 4 — Multi‑Object Tracking

Tracking provides persistent object identity.

Track structure:

- track_id
- age
- hits / misses
- state (tentative / confirmed / lost)
- class histogram
- positions
- bbox sizes
- velocity
- approach features

Lifecycle:

- tentative
- confirmed
- coasting
- terminated

Association methods:

- nearest neighbor
- motion gating
- IoU / center distance
- class consistency

---

<a name="stage-5-temporal-reasoning"></a>
# Stage 5 — Temporal Reasoning

Determines if the object is **approaching**.

Approach indicators:

- increasing bbox area
- stable bearing
- trajectory convergence
- radial motion toward platform

Example rule‑based score:

approach_score =
w1*size_growth +
w2*radial_alignment +
w3*track_persistence +
w4*class_confidence
− w5*motion_chaos

With calibration the system can estimate:

- bearing
- elevation
- angular velocity
- time‑to‑closest approach

---

<a name="stage-6-threat-scoring"></a>
# Stage 6 — Threat Scoring

Combine:

- airborne probability
- drone probability
- track stability
- approach score
- zone relevance
- multi‑target context

Example logic:

- unstable object → low
- confirmed airborne → medium observation
- drone‑like and closing → high
- multiple closing targets → critical alert

Hysteresis prevents alert oscillation.

---

<a name="stage-7-output-and-interface"></a>
# Stage 7 — Output and Interface

Outputs:

- events
- visualization overlays
- ring buffer for event clips
- API for autonomy or operator UI

---

# Tiny Object Strategy

Early detection is fundamentally a **tiny‑object problem**.

Helpful techniques:

- high native resolution
- tiled inference
- avoid aggressive downscaling
- super‑resolution only on crops
- temporal accumulation
- track‑before‑confirm logic

---

# Multi‑Camera Fusion

Fusion improves:

- blind‑spot coverage
- confirmation between cameras
- threat stability
- false‑alarm reduction

Objects seen in two adjacent sectors gain higher confidence.

---

# Key Principle

Success rarely comes from one powerful model.

It comes from the **combination of**:

video conditioning → candidate proposals → crop AI → tracking → temporal reasoning → threat scoring.
