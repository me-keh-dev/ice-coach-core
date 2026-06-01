# Single-Camera 3D Trajectory Reconstruction for Figure Skating: Technical Report

**Version:** 5.0
**Date:** 2026-06-02
**Author:** Yoshihide Tsuruha

---

## Abstract

This document describes the technical approach used by Ice Coach for reconstructing 3D figure skating trajectories from single-camera video. The method requires no camera calibration and estimates on-ice positions, jump trajectories, and skating paths from smartphone-captured video.

---

## 1. Design Philosophy

Ice Coach achieves 3D trajectory reconstruction under the following constraints:

- **No camera calibration**: Processes smartphone video directly without measuring camera intrinsic/extrinsic parameters
- **No rink reference points**: Does not use known coordinate points on the rink
- **Client-side processing**: Runs fully in-browser, no server required

## 2. System Architecture

```
Input: Single-camera video + skater height
       |
       v
[A] Skeletal Pose Estimation
       |  Body keypoints per frame, GPU-accelerated
       |
       +--> [B] Jump Detection (Center of gravity analysis)
       |
       +--> [C] Depth Estimation (Body size in pixels)
       |
       +--> [D] 3D Position Recovery (Pinhole camera model)
       |
       +--> [E] Trajectory Smoothing + Rendering
       |
       v
Output: 3D trajectory + skeleton overlay + 3D rink view
```

## 3. Skeletal Pose Estimation

A pose estimation model detects body keypoints from each video frame. When multiple people are visible, IoU-based tracking follows the target skater across frames.

## 4. Jump Detection

Jumps are detected from the vertical position of the center of gravity. A smoothed vertical signal is computed, and local peaks with sufficient prominence are identified as jump moments. Takeoff and landing frames are determined from the surrounding signal.

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

The "settled" body orientation — the direction the skater faces after completing the landing — is computed as the **average** body facing angle over a time window of **landing + 0.1s to + 0.35s**:

```
settled_angle = mean(facing_angle[t_land + 0.1 ... t_land + 0.35])
```

Averaging over multiple frames (typically 15 frames at 60fps) absorbs the non-determinism inherent in GPU-based pose estimation, which can cause single-frame measurements to vary by 20-40° between runs.

Angles are unwrapped before averaging to handle the ±180° boundary correctly.

**Step 3: Deficit Calculation**

The raw deficit is the angular difference between the body facing at landing and the settled orientation:

```
raw_deficit = |angle_diff(facing_at_landing, settled_angle)|
```

A **checkout offset** of 35° is subtracted to account for normal post-landing rotation on the blade's rocker curve. This rotation is a natural part of the landing check-out and is not considered under-rotation by ISU judges:

```
deficit = max(0, raw_deficit - 35°)
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
| GPU inference non-determinism (20-40° variance per run) | Average settled angle over 0.1-0.35s window |
| Normal post-landing rotation counted as deficit | Subtract 35° checkout offset |
| PC/mobile precision mismatch | Unified heavy model + 60fps on all devices |

### 13.4 Limitations

- Relies on accurate MoCap shoulder positions; body facing toward/away from camera reduces precision
- The 35° checkout offset is empirically determined and may vary across jump types and skaters
- ISU judges assess blade orientation at ice contact; this method measures body (shoulder) orientation as a proxy
- Results may vary ±5° between analysis runs due to video decode and GPU inference non-determinism

---

## References

All methods described in this document are based on well-known physics principles (pinhole camera model, free-fall kinematics) and standard signal processing techniques (median filter, Gaussian smoothing, spline interpolation).

---

*This document was prepared as a technical record of the Ice Coach project.*
