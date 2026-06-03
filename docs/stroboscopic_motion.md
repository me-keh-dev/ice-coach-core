# Automated Stroboscopic Motion Image Generation from Monocular Video: Technical Report

**Version:** 1.0
**Date:** 2026-06-03
**Author:** Yoshihide Tsuruha

---

## Abstract

A stroboscopic motion image (chronophotography) composites sequential poses of a moving subject into a single still image. In figure skating, this visualization is valuable for reviewing jump and spin form at a glance.

Traditionally, creating such images required specialized strobe photography equipment or manual photo retouching. This document presents Ice Coach's fully automated pipeline that generates stroboscopic images from ordinary smartphone video, requiring no special equipment or manual intervention.

The pipeline combines background subtraction, deep learning-based person segmentation, and interactive user editing across seven processing stages.

---

## 1. Prior Art

### 1.1 US 7,042,493 B2 — "Automated stroboscoping of video sequences"

#### Bibliographic Information

| Field | Detail |
|-------|--------|
| **Patent Number** | US 7,042,493 B2 |
| **Title** | Automated stroboscoping of video sequences |
| **Filing Date** | April 6, 2001 |
| **Issue Date** | May 9, 2006 |
| **Assignee** | InMotion Technologies Ltd. |
| **Inventors** | Paolo Prandoni, Emmanuel Reusens, Martin Vetterli, Luciano Sbaiz, Serge Ayer |
| **Status** | **Expired — Lifetime** |
| **Expiration** | April 2021 (20 years from filing) |
| **IPC** | G06T 15/00, H04N 5/262 |
| **Google Patents** | https://patents.google.com/patent/US7042493B2/en |
| **Patent PDF** | https://patentimages.storage.googleapis.com/9a/10/e6/86a882e5ef2357/US7042493.pdf |

#### Technical Summary

A method for automatically generating stroboscopic sequences from standard video footage, including single-camera video of sporting events. The core processing steps are:

1. **Video acquisition**: Input of a video sequence from a single camera
2. **Background estimation**: Construction of a background image from multiple frames using mosaicing techniques
3. **Foreground extraction**: Separation of moving subjects via frame-background differencing
4. **Frame selection**: Automatic selection of frames suitable for the stroboscopic effect
5. **Image composition**: Superposition of extracted foreground onto the background image

Output may be a still image or a video sequence with camera panning effects.

#### Risk Assessment

This patent expired in April 2021 after its full 20-year term under US patent law.

Ice Coach's implementation uses different technical approaches from this patent (compared in Section 5).

---

## 2. Pipeline Overview

```
Input: Monocular video (e.g., smartphone)
       |
       v
[Step 1] Background Generation — Median composition
       |
       v
[Step 2] Motion Detection — Background subtraction + connected component analysis
       |
       v
[Step 3] Target Selection — User taps the person to track on screen
       |
       v
[Step 4] Person Tracking — IoU (Intersection over Union) based tracking
       |
       v
[Step 5] Person Segmentation — Background Matting V2 (BGMv2)
       |                         Cropped to tracked person's bbox
       v
[Step 6] Frame Selection — User selects frames via thumbnail ON/OFF toggle
       |
       v
[Step 7] Alpha Compositing — Placement on background + timestamp label rendering
       |
       v
Output: Stroboscopic composite image (JPEG)
```

---

## 3. Technical Details

### 3.1 Step 1: Background Generation

A clean background image (with no skaters) is automatically generated via median composition from sampled video frames.

Frames are sampled at equal intervals from the video. For each pixel position, the median value across all sampled frames is computed independently for each RGB channel. Since a person occupies any given pixel in only a minority of frames, the median naturally eliminates moving subjects, leaving only the static rink background.

### 3.2 Step 2: Motion Detection

Each frame is compared against the Step 1 background to detect moving people and determine their bounding boxes.

The goal is to obtain person locations (bounding boxes), not detailed skeletal keypoints. Background subtraction was chosen because it requires no additional model loading, keeping the browser-based pipeline lightweight.

#### Difference Mask Generation

For each pixel, the sum of absolute RGB channel differences between the frame and background is computed. Pixels exceeding a threshold are classified as "motion" in a binary mask.

#### Morphological Processing (Closing)

The raw binary mask often fragments a person into separate blobs (head, arms, legs). A closing operation (dilation followed by erosion) merges these fragments into unified body regions.

#### Connected Component Analysis

Flood-fill-based connected component analysis is applied to the closed mask. Each connected region's bounding box (position, size) and area are computed. Regions below a minimum area are discarded as noise.

### 3.3 Step 3: Target Selection

All detected motion regions are highlighted with colored bounding boxes, and the user taps the person they want to track.

During practice sessions, multiple skaters are often visible on the rink, so target selection is left to the user. The initially displayed frame is the one with the most detected people, but the user can navigate to other frames to select from a different time point.

### 3.4 Step 4: Person Tracking

The user-selected person is tracked throughout the video using IoU (Intersection over Union) matching.

#### IoU-Based Frame-to-Frame Matching

IoU measures the overlap between two bounding boxes, ranging from 0 (no overlap) to 1 (perfect match). For each subsequent frame, the IoU between the currently tracked bounding box and all detected motion bounding boxes is computed, and the highest-scoring box is identified as the same person.

#### Bidirectional Tracking

Tracking proceeds in both directions from the user-selected frame — forward in time and backward in time. This ensures that even when the user selects a frame in the middle of the video, the tracking result covers the entire duration.

### 3.5 Step 5: Person Segmentation

For all tracked frames, Background Matting V2 (BGMv2) generates an alpha matte of the skater.

| Field | Detail |
|-------|--------|
| Model | Background Matting V2 (MobileNetV2 backbone) |
| Inference engine | ONNX Runtime Web (runs in browser) |
| Output | Per-pixel alpha matte (continuous values, 0.0–1.0) |

#### BBox-Limited Crop Input

Only a crop around the tracked person's bounding box (with margin) is fed to BGMv2, preventing other skaters from contaminating the mask. BGMv2 is a dual-input model that takes both a source image (frame crop) and a background image (the corresponding region from Step 1), using the difference between them to precisely estimate the person region.

#### Connected Component Separation Within Mask

The BGMv2 output mask may include nearby people within the cropped region. To isolate the target, connected component analysis is applied again within the mask, and only the component closest to the tracked bounding box center is retained. Each component is passed to Step 6 as an individual thumbnail.

### 3.6 Step 6: Frame Selection (Interactive User Editing)

All cutout images generated through the automated steps (Steps 1–5) are presented to the user as a thumbnail gallery. The user selects which frames to include in the final composite by tapping each thumbnail to toggle it ON or OFF.

- The preview image updates in real time as thumbnails are toggled
- "All ON" and "All OFF" buttons allow bulk toggling
- The user downloads the composite once satisfied with the selection

Fully automatic frame selection could not reliably resolve issues such as variable cutout quality and overlapping poses, so the final composition is left to the user's judgment.

### 3.7 Step 7: Alpha Compositing

The user-selected skater cutouts are layered onto the background image to produce the final composite.

#### Alpha Blending

For each pixel, the frame image and background image are linearly interpolated based on the alpha matte value. Edge regions (low alpha) are blended semi-transparently, while the body center (high alpha) is treated as opaque. This produces a natural fusion between the cutout boundaries and the background.

#### Timestamp Labels

Timestamp labels (in seconds) are rendered at each cutout's centroid position. White text with a black outline ensures visibility regardless of the background.

#### Image Output

The final composite is downloadable as a JPEG image. All processing is performed entirely within the browser using the Canvas API — no images are sent to a server.

---

## 4. Output Specification

### Image Output

| Field | Specification |
|-------|---------------|
| Format | JPEG (quality 95%) |
| Image size | Same as input video resolution |
| Content | Background with N skater poses composited at spatially correct positions |

### Metadata Output (JSON)

Per-keyframe data including frame number, timestamp, and joint positions is recorded in JSON format.

---

## 5. Technical Differences from Prior Art

### 5.1 Comparison with US 7,042,493 (2001)

| Processing Step | US 7,042,493 | Ice Coach |
|----------------|--------------|-----------|
| **Motion detection** | Background subtraction (traditional CV) | Background subtraction + connected component analysis |
| **Target selection** | Not described | User taps the person to track on screen |
| **Person tracking** | Not described | IoU-based bidirectional tracking |
| **Foreground extraction** | Frame differencing / chroma key | Background Matting V2 (deep learning alpha matte, continuous values) |
| **Frame selection** | Not detailed | All cutouts presented; user interactively selects via thumbnail toggle |
| **Background generation** | Simple mosaicing | Median composition |
| **Runtime environment** | Not described | Runs entirely in browser (no server required) |

### 5.2 Fundamental Differences

US 7,042,493 is based on image processing techniques available in 2001 (background subtraction, image mosaicing) and contains no deep learning components.

Ice Coach's method uses background subtraction for motion detection (similar to the patent), but employs Background Matting V2 (deep learning alpha matting) for precise person segmentation. The interactive UI design — where users select both the tracking target and final frames — and the fully browser-based runtime environment are also elements not present in the prior patent.

---

## 6. Related Technical Documents

| Document | Content |
|----------|---------|
| [Part 1: 3D Trajectory Reconstruction](technical_report.md) | 3D trajectory, jump height, distance estimation, and under-rotation detection from monocular video. A separate technical module within the same product |
| [Technical Report Index](index.md) | Index of the Ice Coach Technical Report Series |

---

## References

1. **US Patent 7,042,493 B2**: Prandoni, P., Reusens, E., Vetterli, M., Sbaiz, L., Ayer, S. "Automated stroboscoping of video sequences." Filed 2001-04-06, Issued 2006-05-09, Expired 2021-04.
2. **Lin, S. et al.** (2021). "Real-Time High-Resolution Background Matting." *Proceedings of the IEEE/CVF Conference on Computer Vision and Pattern Recognition (CVPR)*.


---

*This document was prepared as a technical record of the Ice Coach project.*
*Patent search based on Google Patents results as of June 3, 2026.*
