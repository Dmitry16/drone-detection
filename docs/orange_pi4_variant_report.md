# Early Airborne Detection on Embedded Compute

<a name="early-airborne-detection-on-embedded-compute"></a>
## Table of Contents
- [Mission and design shift for Orange Pi 4](#1-mission-and-design-shift-for-orange-pi-4)
- [Recommended Orange Pi 4 implementation](#2-recommended-orange-pi-4-implementation)
  - [Sensor and ingest layer](#21-sensor-and-ingest-layer)
  - [Video conditioning](#22-video-conditioning)
  - [Fast candidate generation](#23-fast-candidate-generation)
  - [AI verification on Orange Pi 4](#24-ai-verification-on-orange-pi-4)
  - [Tracking and temporal reasoning](#25-tracking-and-temporal-reasoning)
  - [Practical deployment modes](#26-practical-deployment-modes)
- [Side-by-side architectural comparison](#3-side-by-side-architectural-comparison)
- [Technical limitations of the Orange Pi 4 variant](#4-technical-limitations-of-the-orange-pi-4-variant)
- [Architectural limitations versus Orange Pi 5](#5-architectural-limitations-versus-orange-pi-5)
- [Cost model and total cost comparison](#6-cost-model-and-total-cost-comparison)
- [Recommended decision](#7-recommended-decision)

Orange Pi 4 implementation variant, cost model, and limitations versus
Orange Pi 5

Assumptions used for the cost model are intentionally conservative and
hold the camera payload constant so the board choice is the main
variable.

## 1. Mission and design shift for Orange Pi 4

The original project-source design targets Orange Pi 5-class hardware,
where the main architectural advantage is the RK3588S NPU. The Orange Pi
4 variant must be treated as a CPU-first design. That changes the
implementation philosophy from "NPU-backed heavy path" to "aggressively
pruned fast path with selective verification."

• Keep the same mission: detect very small airborne objects early,
maintain stable tracks, estimate approach, and emit threat-oriented
events.

• Preserve the same layered CV stack from the source materials: ingest →
conditioning → candidate generation → AI verification → tracking →
temporal reasoning → threat scoring.

• Reduce per-frame AI cost aggressively because RK3399 has no onboard
NPU and materially less CPU headroom than RK3588S.

• Prefer 2-camera deployment as the default Orange Pi 4 configuration;
3--4 cameras is possible only with tighter resolution / FPS budgets or
an external accelerator.

## 2. Recommended Orange Pi 4 implementation

The Orange Pi 4 implementation should remain production-style, but it
has to be budgeted differently. The board is viable for an MVP or
constrained fieldable prototype when the pipeline is designed around
cheap motion-based candidate generation and low-cost ROI classification.

### 2.1 Sensor and ingest layer

• Two wide-angle RGB cameras as the baseline. Prefer RTSP/H.264 or USB
cameras so decode remains manageable.

• IMU is still recommended; on Orange Pi 4 it is even more valuable
because better stabilization reduces downstream false positives and
unnecessary AI work.

• Use monotonic timestamps, bounded queues, and isolated workers for
capture, preprocessing, inference, and tracking.

### 2.2 Video conditioning

• IMU-assisted stabilization or global motion estimation between
adjacent frames.

• Light denoise and local contrast normalization; avoid heavy
enhancement that erases tiny targets.

• Sky / skyline / ground masking, with soft masks rather than hard
exclusion, to reduce clutter from trees, roofs, wires, and parallax.

### 2.3 Fast candidate generation

• This becomes the core of the Orange Pi 4 design, not a convenience
stage.

• Use frame differencing after stabilization, local motion saliency,
sparse optical-flow anomalies, and tiny connected-component extraction.

• Only candidates that persist for 2--3 frames, move relative to the
stabilized background, and remain within tiny-target bounds should be
forwarded to AI verification.

### 2.4 AI verification on Orange Pi 4

• Primary option: classifier on crops or a micro-detector on ROI tiles,
running on CPU with NCNN / TFLite-class deployment.

• Do not run a large full-frame detector on every frame from every
camera. On RK3399 this will collapse latency quickly.

• If higher recall is required, add a second-tier confirm stage at a
lower cadence, for example every N frames or only for high-risk tracks.

• For a fieldable v1, classify into drone_like, bird, aircraft,
unknown_airborne, and background_false_alarm.

### 2.5 Tracking and temporal reasoning

• Tracking is non-negotiable because single-frame precision will be
weaker on Orange Pi 4 than on Orange Pi 5.

• Use lightweight Kalman-like prediction, confidence smoothing,
tentative/confirmed/coasting states, and rule-based approach scoring.

• Threat score should combine class confidence, track persistence,
approach features, zone relevance, and multi-object context.

### 2.6 Practical deployment modes

## 3. Side-by-side architectural comparison

Interpretation: the main gap is not only raw CPU speed. The decisive
difference is that Orange Pi 5 can keep the same production pipeline
shape while offloading ROI inference to the NPU, whereas Orange Pi 4
must redesign the compute budget around avoiding inference in the first
place.

## 4. Technical limitations of the Orange Pi 4 variant

No onboard NPU. The Orange Pi 4's largest limitation is the absence of
onboard neural acceleration. In practice this means inference competes
directly with stabilization, tracking, decode, logging, and UI work on
the same CPU budget.

Lower concurrency ceiling. Orange Pi 4 can still ingest and process
multiple streams, but its safe operating envelope is materially
narrower. A cluttered skyline or several simultaneous candidates can
push latency up much faster than on Orange Pi 5.

Weaker tiny-object recall at equal latency. Because the board cannot
afford as much confirm compute, it usually has to trade away either
detector strength, AI cadence, or camera count. That reduces
early-detection confidence for very small and distant objects.

More dependence on handcrafted gating. Masking, thresholds, persistence
rules, and ROI prioritization matter more on Orange Pi 4. That improves
feasibility, but it also makes the system more environment-sensitive and
threshold-sensitive.

Tighter memory headroom. With 3--4 GB-class configurations, buffering,
debug overlays, ring buffers, and multi-camera processing consume a
meaningful share of available memory. This constrains future expansion
more than on Orange Pi 5.

## 5. Architectural limitations versus Orange Pi 5

## 6. Cost model and total cost comparison

To keep the comparison fair, the camera payload is held constant and
only board-specific power / compute choices are changed. The totals
below represent a realistic single-node airborne-detection payload
rather than a bare-board comparison.

Cost delta: about \$77.90 more for the Orange Pi 5 configuration, using
the source report's Orange Pi 5 8GB board price and a current Orange Pi
4 LTS 4GB board price assumption.

Cost interpretation: the Orange Pi 4 variant is cheaper, but the savings
are modest relative to the mission risk introduced by losing onboard NPU
acceleration. If an external AI accelerator is added later, the Orange
Pi 4 node can end up close to or above the Orange Pi 5 node cost while
still remaining architecturally more complex.

## 7. Recommended decision

Choose Orange Pi 4 when: the goal is an MVP, data collection platform,
or low-cost prototype with two cameras and strict ROI budgeting.

Choose Orange Pi 5 when: the goal is a primary onboard production
target, multi-camera deployment, lower latency under clutter, or
stronger tiny-object recall without external accelerators.

Bottom line: Orange Pi 4 can implement the same mission concept, but not
the same compute architecture. It should be treated as a constrained,
CPU-first variant of the original design. Orange Pi 5 remains the better
fit for the full production-style pipeline from the project sources.

### Conclusion (key cost + technical takeaways)

- **Cost comparison (matched 2-camera BOM):**
  - **Orange Pi 4 node:** ~$246.00
  - **Orange Pi 5 node:** ~$323.90
  - **Delta:** ~$77.90
  - Assumes pricing of **$65 for Orange Pi 4 LTS 4GB**, **$140 for Orange Pi 5 8GB**, plus power supplies at **$8.00** and **$10.90**, respectively.

- **Technical takeaways:**
  - Orange Pi 4 is feasible as a **CPU-first, heavily pruned MVP design**, but it cannot preserve the Orange Pi 5 compute architecture one-for-one.
  - The key architectural gap is that RK3399 **lacks onboard NPU hardware**, whereas Orange Pi 5’s RK3588S explicitly advertises an onboard NPU of up to **6 TOPS**.
  - Rockchip’s own RKNN issue tracker confirms that standard RK3399 **does not have NPU hardware**, reinforcing the need for a strictly CPU-budgeted pipeline on Orange Pi 4.

Source basis for this variant: the project-source report and PDF
describing the original Orange Pi 5 solution; official/vendor Orange Pi
product pages for Orange Pi 4 LTS and Orange Pi 5 specifications and
pricing; Rockchip RK3399 and RK3588 family specification material; and
current retailer listings used only for board/power cost assumptions.
