# Single-Camera 3D Trajectory Reconstruction for Figure Skating: Technical Report

**Version:** 5.0
**Date:** 2026-06-02
**Author:** Yoshihide Tsuruha

---

## Abstract

Quantitative analysis of figure skating jumps has traditionally required expensive multi-camera motion capture systems or wearable sensors. This document presents the technical approach used by Ice Coach, a system that performs 3D trajectory reconstruction and automated under-rotation detection from a single smartphone camera — requiring no calibration, no rink reference points, and no wearable sensors.

The key insight is a **2D → 3D → Analysis → 2D pipeline**: 2D video frames are first lifted into 3D Motion Capture data via MediaPipe's World Landmark Model, jump metrics and rotation deficit are computed in 3D world coordinates, and results are projected back onto the original video as overlays. This approach achieves measurement precision that direct 2D image analysis cannot provide.

The under-rotation detection method uses MoCap-derived shoulder orientation with settled-angle averaging and checkout offset compensation.

---

## 1. Design Philosophy

Ice Coach achieves 3D trajectory reconstruction under the following constraints:

- **No camera calibration**: Processes smartphone video directly without measuring camera intrinsic/extrinsic parameters
- **No rink reference points**: Does not use known coordinate points on the rink
- **Client-side processing**: Runs fully in-browser, no server required

## 2. Core Pipeline: 2D → 3D → Analysis → 2D

Ice Coach does not analyze video frames directly in 2D. The processing pipeline is:

1. **2D → 3D Reconstruction**: MediaPipe PoseLandmarker outputs both 2D screen coordinates and 3D World Landmarks (WLM) for each frame. The WLM data (root-relative 3D joint positions in camera space) is transformed into world coordinates by tracking cumulative hip displacement across frames, producing Motion Capture (MoCap) data with absolute positions.

2. **3D Analysis**: All quantitative measurements — jump trajectory, height, under-rotation deficit, biomechanics — are computed from the 3D MoCap data in world coordinates.

3. **3D → 2D Projection**: Analysis results are projected back onto the original 2D video as canvas overlays, synchronized frame-by-frame.

```
Input: Single-camera video + skater height
       |
       v
[A] Skeletal Pose Estimation (MediaPipe PoseLandmarker)
       |  2D keypoints + 3D World Landmarks (WLM), GPU-accelerated
       |
       +--> [B] MoCap Reconstruction (WLM → world coordinates)
       |         -> 3D joint positions per frame
       |
       +--> [C] Jump Detection (Center of gravity analysis)
       |
       +--> [D] Depth Estimation (Body size in pixels)
       |
       +--> [E] 3D Position Recovery (Pinhole camera model)
       |
       +--> [F] Under-Rotation Detection (3D shoulder orientation)
       |
       +--> [G] Trajectory Rendering + Overlay
       |
       v
Output: 3D trajectory + skeleton overlay + under-rotation assessment
```

## 2.1 MoCap Reconstruction

MediaPipe's World Landmark Model outputs root-relative 3D coordinates for each joint (X: lateral, Y: vertical, Z: depth in camera space). These are converted to world coordinates by:

1. **Root tracking**: The pelvis midpoint (average of LeftHip and RightHip) in 2D screen coordinates is tracked frame-to-frame. Screen-space displacement is scaled to world-space displacement.
2. **Joint positioning**: Each joint's WLM root-relative offset is scaled by the skater's known height and added to the world root position.
3. **Ground calibration**: The vertical axis is adjusted so that the lowest foot position corresponds to the ice surface (Y=0).

This produces a per-frame MoCap dataset with world-coordinate joint positions, suitable for biomechanical analysis.

## 3. Skeletal Pose Estimation

MediaPipe PoseLandmarker (heavy model, float16) detects 33 body keypoints per frame using GPU-accelerated (WebGL) inference. The model simultaneously outputs:

- **2D landmarks**: Normalized screen coordinates (x, y) for each joint
- **3D World Landmarks (WLM)**: Root-relative 3D coordinates (x, y, z) in camera space

Video is processed at 60fps regardless of device. When multiple people are visible, IoU-based bounding box tracking follows the target skater across frames, preventing identity switches.

## 4. Jump Detection

Jumps are detected from the vertical position of the center of gravity (CoG), defined as the midpoint between the shoulder center and hip center.

1. **Vertical signal extraction**: The CoG Y-coordinate is extracted for each frame and smoothed to reduce noise.
2. **Peak detection**: Local minima in the vertical signal (where the body reaches maximum height) are identified with sufficient prominence to distinguish jumps from skating movements.
3. **Takeoff detection**: Scanning backward from the peak, the takeoff frame is identified where the CoG begins its upward trajectory.
4. **Landing detection**: Scanning forward from the peak, the landing frame is identified where the CoG returns to its pre-takeoff vertical level.
5. **Airtime measurement**: The duration between takeoff and landing frames provides the airtime T, from which jump height is derived.

## 5. Ice Surface Projection

The ankle's 2D screen position is projected onto the ice surface plane using a ray-plane intersection. Given normalized screen coordinates `(u, v)` and assumed camera parameters (height, tilt angle, focal length):

```
ray_direction = (u/f, -(v/f × cos(tilt) - sin(tilt)), v/f × sin(tilt) + cos(tilt))
t = -camera_height / ray_y
ice_position = t × (ray_x, ray_z)
```

Frame-to-frame displacement on the ice is computed using numerical differentiation (Jacobian):

```
dz/dv ≈ (z(v + ε) - z(v - ε)) / (2ε)
```

This provides the ice-surface trajectory used in the 3D rink view.

## 6. Depth Estimation

Camera-to-skater distance is estimated from the apparent pixel size of the skeleton using a pinhole camera model:

```
Z = L_real × f / L_observed
```

Where:
- `L_real`: Known body segment length in meters (derived from input height and anatomical ratios)
- `f`: Focal length in pixels (estimated from standard smartphone field of view)
- `L_observed`: Observed body segment length in pixels for the current frame

The observed body segment length is pre-processed with a median filter to remove artifacts caused by body rotation.

## 7. 3D Position Recovery

World coordinates are recovered from 2D screen positions and the estimated depth:

```
X = (u - u₀) × Z / f
Y = (v - v₀) × Z / f
```

Where `(u, v)` is the screen position and `(u₀, v₀)` is the principal point. This is a standard pinhole camera back-projection.

## 8. Height Computation

Jump height is determined from airtime using the free-fall physics formula:

```
H = g × T² / 8
```

Where `g` is gravitational acceleration and `T` is the measured airtime. Height is uniquely determined from airtime alone.

## 9. Post-Processing

1. **Median filter**: Removes spike noise from body measurements during rotation
2. **Gaussian smoothing**: Reduces frame-to-frame jitter in on-ice coordinates (applied only to grounded frames)
3. **Curvature constraint**: Enforces minimum turning radius based on skating physics
4. **Velocity clamping**: Limits maximum speed and acceleration to physically plausible values

## 10. Trajectory Rendering

Blade contact points are tracked frame-by-frame and processed through:
1. Median filter for spike removal
2. Moving average for noise reduction
3. Spline interpolation for smooth curve rendering

The rendered trajectory is composited onto the video and synchronized with video frames.

## 11. Scale Calibration

A segmentation model detects the skater's body region. Combined with the input height, this provides a pixel-to-meter conversion factor for real-world measurements.

## 12. Occlusion Handling

- Frames where skeletal estimation fails are marked as invalid and skipped
- Trajectories are connected via interpolation between valid frames
- During detected jump intervals, on-ice position estimation is adjusted accordingly

## 13. Under-Rotation Detection

### 13.1 Overview

Under-rotation detection quantifies the rotational deficit at the moment of jump landing. ISU (International Skating Union) judges assess under-rotation visually, but no standardized computational method has been published. Ice Coach introduces a novel automated approach using MoCap-derived body orientation.

### 13.2 Method

The method uses a 3D Motion Capture (MoCap) reconstruction of the skater's body to measure shoulder orientation on the XZ (ice) plane.

**Step 1: Body Facing Angle**

The body facing direction is computed as the perpendicular to the shoulder line (LeftShoulder → RightShoulder) projected onto the XZ plane of the MoCap world coordinate system:

```
dx = RightShoulder.x - LeftShoulder.x
dz = RightShoulder.z - LeftShoulder.z
facing_angle = atan2(dx, -dz)
```

This captures body rotation in 3D space, avoiding the ±180° flip artifacts that occur with 2D shoulder angle measurements when the body faces toward or away from the camera.

**Step 2: Settled Angle (Reference Orientation)**

The "settled" body orientation — the direction the skater faces after completing the landing — is computed as the **average** body facing angle over a short time window post-landing:

```
settled_angle = mean(facing_angle[t_land + t_start ... t_land + t_end])
```

Averaging over multiple frames absorbs the non-determinism inherent in GPU-based pose estimation.

Angles are unwrapped before averaging to handle the ±180° boundary correctly.

**Step 3: Deficit Calculation**

The raw deficit is the angular difference between the body facing at landing and the settled orientation:

```
raw_deficit = |angle_diff(facing_at_landing, settled_angle)|
```

A **checkout offset** is subtracted to account for normal post-landing rotation on the blade's rocker curve. This rotation is a natural part of the landing check-out and is not considered under-rotation by ISU judges:

```
deficit = max(0, raw_deficit - checkout_offset)
```

**Step 4: ISU Classification**

| Symbol | Deficit Range | GOE Impact |
|--------|--------------|------------|
| OK | < 45° | None |
| q | 45° – 89° | GOE -1 |
| < | 90° – 179° | GOE -2 to -3, BV ×0.80 |
| << | ≥ 180° | GOE -3 to -4, BV one revolution lower |

### 13.3 Design Decisions

| Challenge | Solution |
|-----------|----------|
| 2D heel-toe landmarks too noisy (few pixels) | Use 3D MoCap shoulder orientation instead |
| 2D shoulder angle flips at ±180° during rotation | Use MoCap world coordinates (XZ plane) |
| Camera perspective drift in MoCap hip line | Use shoulder line (more stable during skating) |
| GPU inference non-determinism | Average settled angle over a time window |
| Normal post-landing rotation counted as deficit | Subtract checkout offset |
| PC/mobile precision mismatch | Unified heavy model + 60fps on all devices |

### 13.4 Limitations

- Relies on accurate MoCap shoulder positions; body facing toward/away from camera reduces precision
- The checkout offset is empirically determined and may vary across jump types and skaters
- ISU judges assess blade orientation at ice contact; this method measures body (shoulder) orientation as a proxy
- Results may vary slightly between analysis runs due to video decode and GPU inference non-determinism

---

## 14. Related Work

### Figure Skating Motion Analysis

Prior work on figure skating analysis has primarily relied on professional motion capture systems. Lab-based studies using Vicon or OptiTrack systems have measured jump biomechanics with high precision but require controlled environments and reflective markers, making them impractical for everyday coaching.

### Monocular 3D Pose Estimation

Recent advances in monocular 3D pose estimation (MediaPipe, OpenPose, MMPose) have enabled skeleton detection from single cameras. However, most applications focus on pose recognition or action classification rather than quantitative biomechanical measurement.

### Under-Rotation Assessment

ISU technical panels assess under-rotation through slow-motion video replay using visual judgment. No standardized computational method has been published. Ice Coach provides an automated approach for quantifying landing rotation deficit from single-camera video.

### Positioning of This Work

Ice Coach bridges the gap between lab-grade motion capture and practical coaching tools by:
- Using consumer hardware (smartphone) instead of professional equipment
- Processing entirely client-side (browser) with no server dependency
- Lifting 2D observations into 3D MoCap data for analysis, then projecting results back to 2D

---

## References

The methods described in this document are based on established physics principles (pinhole camera model, free-fall kinematics, projectile motion), standard signal processing techniques (median filter, Gaussian smoothing, spline interpolation), and state-of-the-art pose estimation models (MediaPipe PoseLandmarker).

---

*This document was prepared as a technical record of the Ice Coach project.*
