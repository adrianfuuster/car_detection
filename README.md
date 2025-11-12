# car_detection


Car Tracking and Counting System

Overview

This repository implements a real-time car tracking and counting system built around a modular computer vision pipeline. The goal is to detect vehicles in a video stream, maintain stable identities for each detection across consecutive frames, and increment a running count whenever a vehicle crosses a predefined line or region of interest. The baseline solution relies on classical motion analysis to propose candidate boxes and a centroid-based tracker to preserve identity, while the architecture leaves a clean insertion point for modern deep-learning detectors such as YOLO to replace or augment the detection stage when accuracy in complex scenes is required.

System Architecture

The software is organized as three cooperating layers that communicate through simple, well-defined data structures. The frame reader layer, implemented in reader_frames.py, is responsible for acquiring frames from a file, webcam, or network stream, normalizing resolution and color space as needed, and handing off each frame to the processing loop with consistent timing. The tracking core layer, implemented in CLASS_CentroidTracker.py, encapsulates the identity management logic using a centroid representation; each object is tracked by its geometric center and matched between frames by spatial proximity and motion consistency. The application and coordination layer, implemented in main.py, orchestrates the loop that fetches frames, runs the detector, updates the tracker, applies the counting policy, and renders annotated output for visualization or downstream logging.

This separation enforces a high-cohesion, low-coupling design. The reader focuses purely on input stability, the tracker maintains state and identity persistence, and the coordinator defines the system’s behavior while remaining agnostic to the specific detection backend. As a result, each layer can evolve independently without ripple effects across the rest of the codebase.

Internal Structure and Data Flow

The execution proceeds as a deterministic frame loop. Each iteration begins with frame acquisition from the reader, which returns an image in a predictable format. The detection stage then proposes bounding boxes for candidate vehicles. In the classical configuration this is done via background subtraction or frame differencing followed by contour analysis; in the deep-learning configuration it is handled by a YOLO model that yields class-labeled boxes with confidence scores. The coordinator converts each bounding box into a centroid and forwards the set of centroids to the tracking core.

The tracker’s update routine associates new centroids with existing tracked IDs by minimizing distances between current and previous positions under reasonable motion assumptions. When an existing object cannot be matched in the current frame, its disappearance counter increases; when a new centroid cannot be matched to any existing object, a fresh ID is registered. When an object’s disappearance counter exceeds a threshold, the tracker deregisters it to prevent stale tracks from lingering. Internally the tracker maintains a registry that maps IDs to last-known centroids, a map of disappearance counters keyed by ID, and an integer allocator that assigns the next available ID in strictly increasing order. This lean state makes the tracker predictable, memory-light, and easy to debug.

Once the tracker returns the updated identities and positions, the coordinator evaluates each object trajectory against the counting policy. The most common configuration defines a straight counting line in image coordinates; a crossing event is detected when an object’s centroid is observed on one side of the line in a previous frame and on the opposite side in the current frame, optionally constrained by direction to avoid double counting. The coordinator then overlays bounding boxes, IDs, trajectories if desired, and the current count on the frame and either displays it interactively or forwards it to an encoder or logger.


Code Organization

The repository centers on three Python modules that mirror the architectural layers. The file reader_frames.py encapsulates video capture, resolution control, frame color conversion, and end-of-stream handling, and exposes a simple interface that yields frames to the coordinator. The file CLASS_CentroidTracker.py defines the centroid tracker class with methods to register new objects, update matches for current detections, increment disappearance counters, and deregister objects that have been missing for too long. The file main.py acts as the entry point that wires the reader and tracker together, selects the detection method, maintains the global count, and manages visualization, keyboard control, and optional logging.

Configuration and Parameters

All operational parameters are concentrated in the coordinator so that behavior can be tuned without editing the tracking core. Typical configuration includes the coordinates of the counting line, the disappearance tolerance that controls how many frames an object can vanish before it is removed, and the minimum area or confidence thresholds for detections to suppress noise. When YOLO is used, the coordinator also defines the model variant, input size, confidence and non-maximum suppression thresholds, and the set of class IDs that should be considered vehicles. These parameters are read once at startup and remain constant during the run to ensure reproducible behavior.

Counting Policy and Regions of Interest

The system supports simple line-based counting by default because it is robust and easy to interpret. The line is placed according to the camera viewpoint so that traffic crosses it orthogonally whenever possible, which reduces ambiguous glancing interactions and improves direction discrimination. The same mechanism can be generalized to polygonal regions of interest by tracking whether successive centroid positions fall inside or outside the region; a transition constitutes a counted event. In both cases the policy considers the object’s last-known position to prevent multiple increments for oscillating centroids near the boundary.

Performance Considerations

The baseline motion-based detector is computationally light and suitable for CPUs on laptops or embedded boards, while the YOLO path benefits from GPU acceleration to maintain real-time throughput at higher resolutions. The centroid tracker itself is O(n²) in the number of detections when using greedy matching on distances, but the constant factors are small for typical traffic scenes. Practical deployments benefit from resizing frames to a resolution that preserves license-plate-scale details only if necessary for downstream tasks, otherwise a moderate downscale reduces noise and improves stability without sacrificing counting accuracy.

Testing and Debugging

The modular design simplifies validation. The reader can be tested independently by verifying frame rate and color correctness on diverse inputs. The detector can be validated by visualizing raw boxes before tracking to ensure that proposals are reasonable. The tracker can be stress-tested with synthetic centroid streams to confirm stable ID assignment, disappearance handling, and deregistration timing. The coordinator’s counting logic can be exercised by replaying annotated clips where ground truth crossings are known and checking that the total matches expectations. Because all interfaces are simple Python data structures, unit tests can be written without complex fixtures or mocks.

Limitations and Future Work

The system’s accuracy under heavy occlusion and extreme perspective can be improved by incorporating motion models and appearance descriptors to complement pure centroid distance. The counting policy can be extended to handle multi-lane bidirectional roads by segmenting the scene into lane-specific regions and maintaining per-lane counters. Integration with YOLO can be deepened by class-aware filtering that uses different disappearance tolerances for different vehicle types, and by temporal smoothing of detection confidences. Additional modules for speed estimation, lane adherence, and event logging to databases or message queues would round out a production deployment while keeping the existing architecture intact.
