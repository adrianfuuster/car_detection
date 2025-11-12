# Car Detection & Counting System

## Overview
This repository implements a **real-time car tracking and counting system** built around a modular computer vision pipeline.

The goal is to **detect vehicles** in a video stream, **maintain stable identities** for each detection across consecutive frames, and **increment a counter** whenever a vehicle crosses a predefined line or region of interest.

The baseline solution relies on classical motion analysis to propose candidate boxes and a **centroid-based tracker** to preserve identity, while the architecture leaves a clean insertion point for modern deep-learning detectors such as **YOLO** to replace or augment the detection stage when accuracy in complex scenes is required.

---

## System Architecture
The software is organized as **three cooperating layers** that communicate through simple, well-defined data structures:

1. **Frame Reader Layer** (`reader_frames.py`)  
   - Acquires frames from a file, webcam, or network stream.  
   - Normalizes resolution and color space as needed.  
   - Provides consistent timing to the processing loop.

2. **Tracking Core Layer** (`CLASS_CentroidTracker.py`)  
   - Manages object identities using a centroid representation.  
   - Tracks each object by its geometric center and motion consistency.

3. **Application & Coordination Layer** (`main.py`)  
   - Fetches frames, runs detection, updates the tracker, applies counting policy.  
   - Renders annotated output or forwards it to logs or encoders.

This separation enforces a **high-cohesion, low-coupling design**:  
- The reader focuses on input stability.  
- The tracker maintains identity persistence.  
- The coordinator defines system behavior independently of detection backend.  

Each layer can evolve **independently** without affecting the rest of the system.

---

## Internal Structure & Data Flow
Execution follows a **deterministic frame loop**:

1. **Frame Acquisition:**  
   The reader yields frames in a predictable format.  
2. **Detection Stage:**  
   - Classical mode: background subtraction or contour analysis.  
   - Deep-learning mode: **YOLO** returns class-labeled boxes + confidence scores.
3. **Tracking Stage:**  
   - Converts each bounding box into a centroid.  
   - Associates new centroids with existing IDs by minimizing spatial distance.  
   - Unmatched detections → register new ID.  
   - Unmatched IDs → increase disappearance counter → deregister if too long.
4. **Counting Stage:**  
   - Checks if each trajectory crosses the **counting line or region**.  
   - Increments counter according to direction.

The tracker maintains:
- A registry mapping IDs to centroids.  
- Disappearance counters.  
- An auto-incrementing ID allocator.

Result: **predictable, memory-light, easy to debug.**

---

## Code Organization
| File | Description |
|------|--------------|
| `reader_frames.py` | Handles video capture, resolution control, and frame consistency. |
| `CLASS_CentroidTracker.py` | Implements the centroid-based tracker and ID management. |
| `main.py` | Entry point that wires everything together, runs the loop, and visualizes results. |

---

## Configuration & Parameters
All parameters are centralized in `main.py` for easy tuning:
- Counting line coordinates.
- Disappearance tolerance (frames an object can vanish).
- Minimum area / confidence threshold for detections.
- YOLO configuration:  
  model variant, image size, confidence, NMS threshold, and vehicle class IDs.

Parameters are read once at startup to ensure **reproducible behavior**.

---

## Counting Policy & Regions of Interest
- Default: **Line-based counting** (robust and easy to interpret).  
- The line is placed orthogonally to the traffic direction for clarity.
- Can generalize to **polygonal ROIs**:  
  a transition from inside → outside (or vice versa) counts as an event.

Each crossing event is logged once per object ID to avoid duplicates.

---

## Performance Considerations
- **Classical detector:** lightweight, CPU-friendly.  
- **YOLO mode:** uses GPU for real-time inference.  
- **Centroid tracker:** O(n²) complexity but small constant factors.

**Tips for optimization:**
- Downscale frames moderately to remove noise and keep FPS high.  
- Keep frame size sufficient only for the level of detail required.

---

## Testing & Debugging
Thanks to the modular design, each component can be tested independently:

| Module | Test Focus |
|---------|-------------|
| Reader | Frame rate & color correctness |
| Detector | Visualize boxes before tracking |
| Tracker | Synthetic centroid streams → ID stability |
| Coordinator | Replayed annotated clips → check count accuracy |

Simple **unit tests** can be written easily since all data is pure Python.

---

## Limitations & Future Work
- Under heavy occlusion or perspective distortion, accuracy may drop.  
- Extend to **multi-lane bidirectional roads** (lane-specific counters).  
- Enrich YOLO integration with **class-aware filtering** and smoothing.  
- Add modules for:
  - **Speed estimation**
  - **Lane adherence**
  - **Database logging / message queues**

These can be implemented without changing the existing architecture.

---

## Summary
- **Modular, clean, and real-time** vehicle detection and counting system.  
- Combines **YOLOv8** detection + **Centroid tracking** for identity persistence.  
- Scalable architecture ready for deep learning enhancements.  
- Ideal foundation for traffic analytics, parking monitoring, or smart cities.

---

### Authors
Project: *Car Detection and Counting System*  
Language: Python (OpenCV, NumPy, Ultralytics YOLO)
* **Marc Cases**
* **Álvaro Bello**
* **Namanmahi Kumar**
* **Adrián Fuster**

Universitat Autònoma de Barcelona - 2025


