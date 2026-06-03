# Ice Coach Technical Report Series

**Published by:** Ice Coach Development Team (me-keh-dev)
**Last updated:** 2026-06-03

---

## About This Series

Ice Coach is a system that automatically analyzes figure skating video captured with a smartphone, combining deep learning and physics models. This technical report series documents the design philosophy, algorithms, and implementation details of each technical module that comprises Ice Coach.

---

## Reports

### Part 1: Single-Camera 3D Trajectory Reconstruction for Figure Skating

A method for reconstructing 3D trajectories from a single smartphone camera — no calibration, no rink reference points required. Includes automatic computation of jump height, distance, and velocity, as well as automated under-rotation (landing deficit) detection.

- [English](technical_report.md)
- [日本語](technical_report_ja.md)

**Key technologies:** MediaPipe skeletal pose estimation, pinhole camera model, free-fall physics, MoCap shoulder orientation measurement

---

### Part 2: Automated Stroboscopic Motion Image Generation from Monocular Video

A fully automated pipeline that generates stroboscopic motion images (chronophotography) from ordinary video. Combines deep learning-based skeletal detection, person segmentation, and motion deblurring with figure skating-specific biomechanical knowledge across eight processing stages.

- [English](stroboscopic_motion.md)
- [日本語](stroboscopic_motion_ja.md)

**Key technologies:** Background subtraction, Background Matting V2, ONNX Runtime Web

---

*Each report is self-contained and can be read independently.*
