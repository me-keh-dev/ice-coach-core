# Ice Coach Core

AI-powered figure skating analysis — technical documentation and algorithm design.

## Overview

Technical documentation for the Ice Coach figure skating analysis engine. This repository contains algorithm descriptions and design rationale — **source code is not yet included**.

Ice Coach currently provides two analysis modes:

### Jump Analysis: 2D → 3D → Analysis → 2D

Ice Coach does not analyze video frames directly in 2D. Instead:

1. **2D → 3D**: MediaPipe PoseLandmarker extracts both 2D skeleton and 3D World Landmarks (WLM) from each frame. The WLM data is then converted into Motion Capture data in world coordinates.
2. **3D Analysis**: Jump trajectory, under-rotation, and biomechanics are computed from the 3D MoCap data — achieving precision that 2D analysis alone cannot provide.
3. **3D → 2D**: Results are projected back onto the original 2D video as overlay visualizations.

### Spin Analysis: 2D → Fourier → Ice Trajectory

Spin trajectory reconstruction separates the skater's actual drift (traveling) from rotational oscillation using Fourier decomposition at the known rotation frequency. Shadow detection on the ice surface provides independent drift correction.

Both pipelines run entirely in the browser from a single smartphone camera, with no server, no calibration, and no rink reference points.

## Key Features

- **Zero calibration**: No camera intrinsic/extrinsic parameter measurement needed
- **No rink reference points**: Works without any known coordinates on the ice surface
- **Single smartphone camera**: No multi-camera setup or wearable sensors
- **Runs in browser**: Fully client-side, no server required
- **Physics-grounded**: Jump height derived from airtime using kinematic equations
- **Fourier-based spin trajectory**: Rotation/drift separation for traveling detection

## How It Works

### Jump Analysis Pipeline

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

### Spin Analysis Pipeline

```
Input: Single-camera spin video (stationary camera)
       |
       v
[A] Skeletal Pose Estimation (MediaPipe Pose)
       |
       +--> [B] Rotation Speed (Shoulder angle → RPS)
       |
       +--> [C] Spin Detection (Hip movement + body width oscillation)
       |
       +--> [D] Axis Foot (Multi-frame voting, locked for duration)
       |
       +--> [E] Blade Contact Tracking (Heel-toe midpoint)
       |
       +--> [F] Fourier Decomposition (Rotation/drift separation)
       |         -> DC component = traveling, 1st harmonic = rotation
       |
       +--> [G] Shadow-Based Drift Correction
       |         -> Shadow stationary = no traveling (clamp drift to zero)
       |
       +--> [H] Axis Stability Scoring (COG offset + trunk tilt)
       |
       v
Output: Ice trajectory map + minimap overlay + axis stability piano roll
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

---

## Spin Analysis: Technical Approach

### The Problem: Rotation and Translation Are Entangled

When a skater spins, the skating foot's 2D position oscillates left and right with each revolution. This oscillation is not actual movement — it is the projection of circular motion onto the camera's image plane. The skater's real drift (traveling) is hidden within this oscillation.

### Solution: Fourier Decomposition at Known Frequency

The skating foot's 2D position is modeled as:

```
x(t) = center_x(t) + radius × cos(theta(t))
```

Where `theta(t)` is the cumulative rotation angle, already known from shoulder angle analysis. Fourier decomposition at the rotation frequency separates the DC component (drift) from the first harmonic (rotation).

```
center_x = (1/N) × Σ x[j]                    (drift)
A_x = (2/N) × Σ x[j] × cos(theta[j])         (rotation cosine)
B_x = (2/N) × Σ x[j] × sin(theta[j])         (rotation sine)
radius_x = sqrt(A_x² + B_x²)
```

### Shadow-Based Drift Correction

Posture changes (e.g., sit spin entry) can shift the Fourier center even when the skater is stationary, producing false drift. The skater's shadow on the ice surface provides ground-truth position: if the shadow doesn't move, the skater isn't traveling. When shadow range is below a threshold, drift is clamped to zero.

The shadow also provides the camera elevation angle (`sin(alpha) = shadow_length / body_height`), used for top-down trajectory reconstruction.

### Axis Stability Scoring

A combined metric integrating:
- **COG offset**: Distance between full-body center of gravity and axis foot, using anatomical segment mass ratios
- **Trunk tilt**: Horizontal displacement of the trunk midline from the axis foot vertical
- **Centrifugal tolerance**: Adjusted offset allowance at higher rotation speeds

### Axis Foot Determination

Multi-frame voting at spin start, locked for the entire spin duration. Single-frame Y-coordinate comparison fails during sit spins where both feet appear at similar heights.

### Visualization

- **Trajectory map**: SVG below video, color-coded by axis stability, with Catmull-Rom spline interpolation
- **Video minimap**: Circular overlay in bottom-left corner with past/future trajectory and real-time position dot
- **Piano roll**: Multi-track axis stability score synchronized with video playback

---

## Documentation

| Document | Description |
|----------|-------------|
| [Jump Analysis Technical Report](docs/technical_report.md) | Full technical details of the jump analysis pipeline |
| [Jump Analysis Technical Report (Japanese)](docs/technical_report_ja.md) | 日本語版 |
| [Spin Analysis Technical Report](docs/spin_analysis_technical_report.md) | Full technical details of the spin trajectory reconstruction |
| [Spin Analysis Technical Report (Japanese)](docs/spin_analysis_technical_report_ja.md) | 日本語版 |

## License

MIT License. See [LICENSE](LICENSE) for details.

## Author

**Yoshihide Tsuruha**

25+ years in IT/AI engineering. Building Ice Coach — 3D figure skating analysis from a single camera.

## Contact

For questions, bug reports, feature requests, or research collaboration, please open a [GitHub Issue](https://github.com/me-keh-dev/ice-coach-core/issues).
