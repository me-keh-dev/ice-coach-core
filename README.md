# Ice Coach Core

AI-powered figure skating jump analysis — technical documentation and algorithm design.

## Overview

Technical documentation for the Ice Coach figure skating analysis engine. This repository contains algorithm descriptions and design rationale — **source code is not yet included**.

### Core Pipeline: 2D → 3D → Analysis → 2D

Ice Coach does not analyze video frames directly in 2D. Instead:

1. **2D → 3D**: MediaPipe PoseLandmarker extracts both 2D skeleton and 3D World Landmarks (WLM) from each frame. The WLM data is then converted into Motion Capture data in world coordinates.
2. **3D Analysis**: Jump trajectory, under-rotation, and biomechanics are computed from the 3D MoCap data — achieving precision that 2D analysis alone cannot provide.
3. **3D → 2D**: Results are projected back onto the original 2D video as overlay visualizations.

This 2D→3D→2D pipeline is the key to extracting accurate measurements from ordinary smartphone video.

Built from the ground up for real-world use on the ice rink, this engine combines skeletal pose estimation, physics-based jump analysis, and anthropometric depth estimation to produce 3D motion data from ordinary video.

## Key Features

- **Zero calibration**: No camera intrinsic/extrinsic parameter measurement needed
- **No rink reference points**: Works without any known coordinates on the ice surface
- **Single smartphone camera**: No multi-camera setup or wearable sensors
- **Runs in browser**: Fully client-side, no server required
- **Physics-grounded**: Jump height derived from airtime using kinematic equations

## How It Works

```
Input: Single-camera video
       |
       v
[A] Skeletal Pose Estimation (MediaPipe Pose)
       |
       +--> [B] Horizontal Position (Ice surface projection + Jacobian differentials)
       |         -> 2D ice trajectory (x, y)
       |
       +--> [C] Jump Detection (CNN classifier)
       |         -> Takeoff/landing frames, airtime T
       |
       +--> [D] Jump Height (Physics formula)
       |         -> h(t) = (g/2) * t * (T - t)
       |
       +--> [E] Depth Estimation (Body size ratio)
       |         -> Camera distance (estimate)
       |
       +--> [F] Under-Rotation Detection (MoCap shoulder orientation)
                 -> ISU deficit classification (OK/q/</<<)
       |
       v
Output: 3D trajectory + velocity + distance + under-rotation assessment
```

## Technical Approach

### Horizontal Position: Ground-Contact Projection + Jacobian Differentials

Ankle positions in image coordinates are projected onto the ice surface using a fixed-assumption camera model. Frame-to-frame displacement is computed via Jacobian differentials.

### Jump Height: Direct Physics Calculation

Jump height is uniquely determined from airtime T alone:

```
H_max = g * T^2 / 8
h(t) = (g/2) * t * (T - t)
```

Height is uniquely determined from airtime alone.

### Depth Estimation: Anthropometric Body Size Ratio

Camera-to-skater distance is estimated by comparing the pixel size of body segments (e.g., spine length) against well-established anatomical body proportions.

```
depth = spine_real_meters * focal_px / spine_pixels
```

### Under-Rotation Detection: MoCap Shoulder Orientation

A novel method for quantifying jump landing under-rotation using 3D body orientation.

**Body Facing Angle**: The perpendicular to the shoulder line (LeftShoulder → RightShoulder) is projected onto the XZ (ice) plane of the MoCap world coordinate system:

```
facing_angle = atan2(dx_shoulder, -dz_shoulder)
```

**Settled Angle**: The reference body orientation after the skater stabilizes, computed as the average facing angle over a short time window post-landing. Averaging over multiple frames absorbs GPU inference non-determinism.

**Deficit Calculation**:

```
raw_deficit = |angle_diff(facing_at_landing, settled_angle)|
deficit = max(0, raw_deficit - checkout_offset)
```

The checkout offset accounts for normal post-landing rotation on the blade's rocker curve, which ISU judges do not count as under-rotation.

**ISU Classification**:

| Symbol | Deficit | Impact |
|--------|---------|--------|
| OK | < 45° | None |
| q | 45°–89° | GOE -1 |
| < | 90°–179° | GOE -2 to -3, BV ×0.80 |
| << | ≥ 180° | GOE -3 to -4, BV one revolution lower |

### Trajectory Rendering: Physics-Based Jump Arc

- **Ground frames**: Blade edge position (heel/toe lowest point + ankle offset)
- **Airborne frames**: Parabolic arc computed from takeoff/landing positions — no noisy foot landmarks used during flight
- **Spike removal**: Physics-constrained maximum velocity per frame + neighbor deviation outlier detection

### Post-Processing: Physics Constraints

- **Airborne smoothing**: Horizontal coordinates are smoothly interpolated during the jump phase
- **Gaussian smoothing**: Noise reduction for on-ice XZ coordinates
- **Curvature constraint**: Minimum turning radius based on skating physics
- **Velocity clamping**: Maximum speed and acceleration limited to physically plausible values

## License

MIT License. See [LICENSE](LICENSE) for details.

## Author

**Yoshihide Tsuruha**

25+ years in IT/AI engineering. Building Ice Coach — 3D figure skating analysis from a single camera.

## Contact

For questions, bug reports, feature requests, or research collaboration, please open a [GitHub Issue](https://github.com/me-keh-dev/ice-coach-core/issues).
