# Single-Camera Spin Trajectory Reconstruction for Figure Skating: Technical Report

**Version:** 3.0
**Date:** 2026-06-20
**Author:** Yoshihide Tsuruha

---

## Abstract

Quantitative assessment of spin traveling in figure skating has traditionally relied on coaches' visual judgment or expensive multi-camera systems. This document presents the technical approach used by Ice Coach's spin analysis module, which reconstructs an approximate ice-surface trajectory from a single stationary smartphone camera — requiring no calibration, no rink reference points, and no server processing.

The key insight is that the skating foot's 2D oscillation during rotation has a **known frequency** (the rotation speed, already measured from shoulder angle analysis), enabling **Fourier decomposition** to separate the skater's actual drift from the rotational component. Shadow detection on the ice surface provides an independent correction for false drift caused by posture changes.

---

## 1. Design Philosophy

The spin analysis module operates under the same constraints as the jump analysis:

- **No camera calibration**: Processes smartphone video directly without measuring camera intrinsic/extrinsic parameters
- **No rink reference points**: Does not use known coordinate points on the rink
- **Client-side processing**: Runs fully in-browser using MediaPipe and Canvas API, no server required
- **Stationary camera**: Assumes a fixed camera position (tripod or rink-side placement)

## 2. Core Pipeline

```
Input: Single-camera spin video + skater height
       |
       v
[A] Skeletal Pose Estimation (MediaPipe PoseLandmarker, heavy model)
       |  33 keypoints per frame, GPU-accelerated
       |
       +--> [B] Rotation Speed Estimation (Shoulder angle → RPS)
       |
       +--> [C] Spin Segment Detection (Hip movement + body width)
       |
       +--> [D] Axis Foot Determination (Multi-frame voting)
       |
       +--> [E] Blade Contact Point Tracking (Heel-toe midpoint)
       |
       +--> [F] Noise Filtering (RPS threshold, jump distance, off-axis)
       |
       +--> [G] Fourier Decomposition (Rotation/drift separation)
       |
       +--> [H] Shadow-Based Drift Correction
       |
       +--> [I] Camera Elevation Estimation (Fourier amplitude ratio)
       |
       +--> [J] Trajectory Smoothing + Rendering
       |
       v
Output: Ice-surface trajectory map + video minimap overlay + axis stability score
```

## 3. Rotation Speed Estimation

### 3.1 Shoulder Angle Method

Rotation speed (RPS: revolutions per second) is estimated from the frame-to-frame change in shoulder angle:

```
sa = atan2(shoulder_R.y - shoulder_L.y, shoulder_R.x - shoulder_L.x)
d = sa[i] - sa[i-1]     (with ±180° wrapping)
rps = |d| × fps / 360
```

Using MediaPipe shoulder landmarks. A temporal moving average suppresses frame-to-frame noise.

### 3.2 2D Projection Correction

In 2D video, the projected shoulder angle oscillates **twice per revolution** (maximum width at both front-facing and back-facing orientations). This creates a systematic 2× overestimation in revolution counting. A correction factor is applied at the aggregation level to compensate.

### 3.3 Cumulative Rotation Angle

The cumulative rotation angle, required for Fourier decomposition, is computed by integrating the per-frame RPS:

```
theta[i] = theta[i-1] + rps[i] / fps × 2π
```

## 4. Spin Segment Detection

### 4.1 Three-Stage Detection

**Stage 1 — Raw segment detection**: A sliding window identifies frames where:
- Hip midpoint movement remains below a threshold (skater remains roughly stationary)
- Body width standard deviation exceeds a threshold (body silhouette oscillates, indicating rotation)

**Stage 2 — Segment merging**: Short gaps between detected segments are merged, accommodating position changes within combination spins.

**Stage 3 — Minimum threshold**: Segments must exceed minimum duration and revolution count to be classified as valid spins.

## 5. Axis Foot Determination

### 5.1 Multi-Frame Voting

The skating (axis) foot cannot be reliably determined from a single frame, particularly during sit spins where both feet may appear at similar heights. The system uses a voting mechanism:

1. During the initial frames of a detected spin, each frame votes for which foot is the axis foot based on Y-coordinate (larger Y = closer to ice surface)
2. The majority vote is locked for the entire spin duration
3. The determination is stored per-frame for consistent rendering

### 5.2 Ankle Position Smoothing

Even with correct foot selection, the ankle landmark position fluctuates frame-to-frame due to pose estimation noise. An exponential moving average (EMA) smooths the axis foot's X-coordinate for stable visualization.

### 5.3 Abnormal Frame Filtering

Frames where both ankles are significantly above the body's lowest point are classified as pose estimation errors and excluded from rendering. This filters out frames where MediaPipe produces physically impossible skeleton configurations during fast rotation.

## 6. Blade Contact Point Tracking

For each valid frame during the spin, the blade contact point is computed as:

```
blade_x = (heel.x + toe.x) / 2
blade_y = (heel.y + toe.y) / 2 + body_height × offset_ratio
```

Using MediaPipe heel and toe landmarks. The downward offset compensates for the distance between the ankle joint and the blade's ice contact point.

## 7. Noise Filtering

Three filters remove tracking noise before trajectory reconstruction:

1. **RPS threshold**: Frames below a minimum rotation speed are discarded (skater is not yet spinning or has stopped)
2. **Jump distance filter**: Frames where the foot position jumps beyond a threshold from the previous frame are discarded (landmark tracking failure)
3. **Off-axis detection**: A running median of foot X-position is maintained. Sustained deviation from the median triggers spin-end detection

## 8. Fourier Decomposition

### 8.1 Signal Model

The skating foot's 2D position during a spin is modeled as:

```
x(t) = center_x(t) + radius × cos(theta(t))
y(t) = center_y(t) + radius × sin(theta(t)) × sin(alpha)
```

Where:
- `center(t)` = actual ice-surface position (drift component)
- `radius` = distance from rotation axis to blade contact point
- `theta(t)` = cumulative rotation angle (known from Section 3.3)
- `alpha` = camera elevation angle

### 8.2 Sliding-Window Decomposition

Over a window of approximately 1 revolution centered at each sample point:

```
center_x = (1/N) × Σ x[j]                    (DC component = drift)
A_x = (2/N) × Σ x[j] × cos(theta[j])         (Fourier cosine coefficient)
B_x = (2/N) × Σ x[j] × sin(theta[j])         (Fourier sine coefficient)
radius_x = sqrt(A_x² + B_x²)                  (rotation amplitude)
```

The rotation radius is capped to prevent free leg oscillation from contaminating the estimate.

### 8.3 Y-Axis Stabilization

The vertical drift component (`center_y`) is fixed to the median value across the entire spin. Vertical variation in 2D is caused by posture changes (sit vs. upright position), not actual movement on the ice surface.

### 8.4 Top-Down Reconstruction

The ice-surface trajectory is reconstructed by combining the separated drift and rotation components:

```
x_ice = center_x + radius_x × cos(theta)
z_ice = radius_x × sin(theta)
```

The Z-axis uses only the X-amplitude rotation radius rather than `center_y / sin(alpha)`, because the Y-center is unreliable due to posture-induced vertical shifts.

## 9. Shadow-Based Drift Correction

### 9.1 Problem

Posture changes (e.g., transitioning from upright to sit spin) can shift the Fourier center position even when the skater remains stationary. This produces false drift in the reconstructed trajectory.

### 9.2 Shadow Detection

The skater's shadow on the ice surface provides ground-truth position information:

1. Select a standing frame before the spin begins
2. Detect the foot position and body bounding box using MediaPipe
3. Apply preprocessing to enhance shadow contrast
4. Mask the body bounding box region
5. In the ice surface below the feet, identify dark regions as shadow using adaptive thresholding
6. Apply morphological operations for noise removal

### 9.3 Drift Clamping

If the shadow's horizontal range during the spin is below a threshold, the drift is clamped to zero for all Fourier fits. The shadow confirms the spin is stationary, and any apparent center drift is an artifact.

### 9.4 Camera Elevation Estimation

The shadow provides an independent estimate of the camera elevation angle:

```
sin(alpha) = shadow_length / body_height
```

This is cross-referenced with the Fourier amplitude ratio (`avg_ry / avg_rx`). The shadow-derived value takes priority when available, as it is more physically grounded.

## 10. Axis Stability Scoring

### 10.1 Combined Metric

The axis stability score integrates two components into a single quality metric:

**Component 1 — COG offset**: Distance between the full-body center of gravity and the axis foot position, normalized by frame width. The COG is computed using anatomical segment mass ratios weighted across head, trunk, arms, and legs.

**Component 2 — Trunk tilt**: Horizontal displacement of the trunk midline (shoulder midpoint to hip midpoint) from the axis foot vertical.

**Centrifugal tolerance**: At higher RPS, a small additional offset tolerance is applied to account for expected centrifugal displacement.

### 10.2 Scoring

The combined offset is classified into alignment levels (aligned / good / poor) with corresponding color coding. Thresholds are scaled by user-selected difficulty level to accommodate different skill levels.

## 11. Trajectory Smoothing and Rendering

### 11.1 Post-Processing

The final trajectory points are smoothed with a moving average to eliminate residual noise from pose estimation.

### 11.2 Trajectory Map

Displayed below the video as an SVG element:
- Line color encodes axis stability score per segment
- Catmull-Rom spline interpolation for smooth curve rendering
- Current playback position highlighted and synchronized with video

### 11.3 Video Minimap Overlay

The same trajectory data is rendered as a circular minimap in the bottom-left corner of the video canvas:
- Past trajectory shown in a bright color
- Future trajectory shown dimmed
- Current position marked with a dot
- Updates in real-time during video playback

## 12. Limitations

1. **Single-camera depth ambiguity**: The Z-axis (depth into ice) is estimated from the Fourier Y-amplitude ratio, which is imprecise for cameras at near-horizontal angles
2. **RPS estimation dependency**: Trajectory accuracy depends on the RPS estimation from 2D shoulder angles, which can be noisy during certain spin positions (particularly sit spins where shoulders are partially occluded)
3. **Not metric-scale**: Without camera calibration, the trajectory shows relative movement patterns but not absolute distances in meters
4. **Stationary camera only**: The current release assumes a fixed camera position
5. **Posture-dependent noise**: Despite shadow correction, rapid posture changes can still introduce small trajectory artifacts

## 13. Validation

The method was validated against multiple test videos with varying spin types (single position, combination) and camera conditions (fixed tripod, rink-side). Qualitative assessment by a figure skating coach confirmed that the reconstructed trajectory shapes correspond to observed traveling patterns.

---

## 14. Related Work

### Figure Skating Spin Analysis

Prior work on spin assessment has primarily relied on professional motion capture systems or multi-camera setups. Lab-based studies using marker-based systems have measured spin biomechanics with high precision but require controlled environments and specialized equipment.

### Monocular Pose Estimation for Sports

Recent advances in monocular pose estimation have enabled skeleton detection from single cameras. Applications in sports analysis have focused on action recognition and pose classification rather than trajectory reconstruction.

### Positioning of This Work

Ice Coach's spin trajectory reconstruction addresses a gap in existing tools by:
- Using consumer hardware (smartphone) instead of professional equipment
- Separating rotation from drift using signal processing (Fourier decomposition) rather than multi-camera geometry
- Leveraging environmental cues (ice surface shadows) for independent drift validation
- Processing entirely client-side (browser) with no server dependency

---

## References

The methods described in this document are based on established physics principles (circular motion projection, Fourier analysis), standard signal processing techniques (sliding-window decomposition, median filtering, moving average), computer vision methods (adaptive histogram equalization, morphological operations), and state-of-the-art pose estimation models.

---

*This document was prepared as a technical record of the Ice Coach project's spin analysis module. All processing runs client-side in the user's browser.*
