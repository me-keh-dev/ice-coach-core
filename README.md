# Ice Coach Core

Single-camera 3D trajectory reconstruction engine for figure skating analysis.

## Overview

Ice Coach Core is an open-source engine that reconstructs 3D skating trajectories from a single smartphone camera — **no calibration, no reference points, no sensors required**.

Built from the ground up for real-world use on the ice rink, this engine combines skeletal pose estimation, physics-based jump analysis, and anthropometric depth estimation to produce 3D motion data from ordinary video.

## Key Features

- **Zero calibration**: No camera intrinsic/extrinsic parameter measurement needed
- **No rink reference points**: Works without any known coordinates on the ice surface
- **Single smartphone camera**: No multi-camera setup or wearable sensors
- **Runs in browser**: Fully client-side, no server required
- **Physics-grounded**: Jump height derived from airtime using kinematic equations — not image regression

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

No image-based inverse projection is used. No optimization is performed.

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

**Settled Angle**: The reference body orientation after the skater stabilizes, computed as the average facing angle over a 0.1–0.35s window post-landing:

```
settled = mean(facing_angle[t_land + 0.1 ... t_land + 0.35])
```

Averaging over ~15 frames absorbs GPU inference non-determinism (which can cause 20–40° variance in single-frame measurements).

**Deficit Calculation**:

```
raw_deficit = |angle_diff(facing_at_landing, settled_angle)|
deficit = max(0, raw_deficit - 45°)
```

The 45° offset accounts for normal post-landing rotation on the blade's rocker curve (checkout), which ISU judges do not count as under-rotation.

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

- **Airborne linear motion**: No external forces act horizontally during a jump, so XZ coordinates are linearly interpolated between takeoff and landing
- **Gaussian smoothing**: Noise reduction for on-ice XZ coordinates
- **Curvature constraint**: Minimum turning radius 0.75m
- **Velocity clamping**: Max speed 8.4 m/s, max acceleration 5.0 m/s^2

## License

MIT License. See [LICENSE](LICENSE) for details.

## Author

**Yoshihide Tsuruha**

25+ years in IT/AI engineering. Building Ice Coach — 3D figure skating analysis from a single camera.

## Contact

For questions, bug reports, feature requests, or research collaboration, please open a [GitHub Issue](https://github.com/me-keh-dev/ice-coach-core/issues).
