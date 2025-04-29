# Face-Tracked 3D Window

**[➡️ Live Demo (GitHub Pages) ⬅️](https://konradish.github.io/magic-window/)**

**tl;dr** – This page spins up MediaPipe FaceMesh to grab eye landmarks from your webcam, pipes them into a tiny math shim, and offsets a THREE.js perspective-camera so the cube in front of you parallax-shifts as your head moves. Open the file over HTTPS, allow camera access, and your monitor turns into a faux “window” on a 3-D scene. Zero external servers are involved; everything runs client-side.

## How it Works (Updated Logic)

1.  **Webcam + Face Tracking**
    *   [MediaPipe FaceMesh](https://github.com/google/mediapipe/blob/master/mediapipe/modules/face_geometry/README.md) runs in WebAssembly and streams 468 facial-landmark points in real-time (≈15 ms on a mid-tier laptop). ([jsDelivr CDN](https://www.jsdelivr.com/package/npm/@mediapipe/face_mesh))
    *   The helper [@mediapipe/camera\_utils](https://www.npmjs.com/package/@mediapipe/camera_utils) ([GitHub](https://github.com/google/mediapipe/tree/master/mediapipe/web/solutions/camera_utils)) automagically handles `getUserMedia`, pumps frames into the model, and calls back every render tick.
    *   Landmark indices `468` (right pupil center) and `473` (left pupil center) are used for stable head position and distance estimation. ([Reference](https://github.com/tensorflow/tfjs-models/blob/master/face-landmarks-detection/mesh_map.jpg))

2.  **Mapping Head Pose → Virtual Camera**
    *   **Physical Setup:** You provide your monitor's physical width and height (in mm) and optionally your Inter-Pupillary Distance (IPD).
    *   **Head Position (X/Y):** The midpoint between the detected pupils (in normalized video coordinates 0-1) is mapped to a physical position relative to the screen center (in meters). Moving your head left/right/up/down moves the virtual camera's X/Y position accordingly.
    *   **Head Distance (Z):**
        *   The distance between your pupils is measured in pixels (`eyeDistPx`) in the video feed.
        *   You calibrate by clicking "Set Near Point" (when close) and "Set Far Point" (when far). This records the `eyeDistPx` for these two extremes.
        *   The current `eyeDistPx` is then mapped (linearly interpolated) between the calibrated near/far pixel distances to estimate your current physical distance from the screen (Z position in meters).
    *   **Smoothing:** An Exponential Moving Average (EMA) filter smooths the calculated X, Y, and Z target positions to reduce jitter. The smoothing factor is adjustable.
    *   **Parallax Scale:** An adjustable slider allows exaggerating the X/Y camera movement for artistic effect.

3.  **View-Dependent Rendering**
    *   [THREE.js](https://threejs.org/) (via [cdnjs CDN](https://cdnjs.com/libraries/three.js/)) renders the 3D scene.
    *   **Dynamic FOV:** The virtual camera's Field of View (FOV) is calculated *every frame* based on the current estimated head distance (Z) and the physical screen height. This ensures the perspective projection matches your viewpoint, making the screen appear like a true window. Formula: `fov = 2 * atan( (screenHeight / 2) / headDistance )`.
    *   **Projection Update:** The camera's `projectionMatrix` is updated each frame to reflect the changes in position and FOV.
    *   **Look At:** The camera always looks towards the center of the scene (0,0,0).

## Quick Start

> **heads-up:** browsers expose `getUserMedia()` only from a secure context (https or `http://localhost`). if you just double-click the html file and see a grey box with no permission prompt, that’s the issue. [[MDN](https://developer.mozilla.org/en-US/docs/Web/API/Navigator/getUserMedia)]

1.  **Serve over HTTPS:** Browsers (like Chrome) block webcam access on `file://` URLs for security reasons. The simplest way to serve the `index.html` file locally over HTTPS is using `npx`:
    ```bash
    npx serve .
    ```
    Then open the HTTPS URL provided (usually `https://localhost:3000` or similar). Allow camera access when prompted.
    *Alternatively, use Python and a service like nip.io for a valid HTTPS certificate on localhost:*
    ```bash
    python -m http.server 8080
    ```
    *Then navigate to `https://127.0.0.1.nip.io:8080` in your browser.*

2.  **Open `index.html`** in your browser via the HTTPS URL.

3.  **Allow camera access** when prompted by the browser.

4.  **Configure Physical Dimensions:**
    *   Measure the **visible width and height** of your monitor screen in millimeters (e.g., with a ruler).
    *   Enter these values into the "Screen Width (mm)" and "Screen Height (mm)" input boxes. Defaults are provided for a ~15" screen.
    *   (Optional) Adjust the "IPD (mm)" value if you know yours (average is ~63mm). This isn't critical for the current depth calculation but might be used in future refinements.

5.  **Calibrate Depth:**
    *   You'll see buttons "Set Near Point", "Set Far Point", and "Reset Calibration" at the bottom.
    *   **Lean fully towards your screen** to the closest comfortable distance you want tracked. Click **"Set Near Point"**.
    *   **Lean back** to the furthest comfortable distance you want tracked. Click **"Set Far Point"**.
    *   The status text will update. If it shows "Calibrated", the depth tracking is active. If it shows an error (e.g., Near Px <= Far Px), click "Reset Calibration" and try again.

5.  **Move your head:**
    *   Move side-to-side and up/down: The scene shifts like looking through a window.
    *   Lean in/out (after calibration): The camera should zoom in/out, enhancing the 3D effect.

## Extensibility Ideas

| Tweak                 | What to change                                                                                                |
| :-------------------- | :------------------------------------------------------------------------------------------------------------ |
| Use IPD for Z       | Implement Z calculation: `headZmm = ipdMM * focalPx / eyeDistPx`. Requires estimating webcam `focalPx`. |
| Improve Landmarks   | Average multiple pupil/iris landmarks (e.g., `468-472` left, `473-477` right) for more robust center points. |
| Better Smoothing    | Swap simple EMA for a [One-Euro Filter](https://cristal.univ-lille.fr/~casiez/1euro/) or Kalman filter for lower latency smoothing. |
| Richer scene          | Load a GLTF model using `THREE.GLTFLoader`; add post-processing effects.                                      |
| Fallback Controls   | Add mouse/touch controls that activate when face tracking is lost (see munrocket/parallax-effect). |

## Dependencies & References

*   **MediaPipe FaceMesh:**
    *   [CDN Link (jsDelivr)](https://www.jsdelivr.com/package/npm/@mediapipe/face_mesh)
    *   [Official Docs (Landmarks, Config)](https://developers.google.com/mediapipe/solutions/vision/face_landmarker/index#face_landmarks)
    *   [JS Quick Start Guide (Camera Utils)](https://developers.google.com/mediapipe/solutions/vision/face_landmarker/web_js)
*   **@mediapipe/camera\_utils:**
    *   [npm Package](https://www.npmjs.com/package/@mediapipe/camera_utils)
    *   [GitHub Source](https://github.com/google/mediapipe/tree/master/mediapipe/web/solutions/camera_utils)
    *   [Camera Utils Deep Dive (GitHub Issue)](https://github.com/google/mediapipe/issues/1220)
*   **Landmark Mapping:**
    *   [Blog Post (Selvam.tech)](https://www.selvam.tech/blogs/mediapipe-facemesh-keypoints/)
    *   [Stack Overflow Discussion](https://stackoverflow.com/questions/66979377/mediapipe-face-mesh-keypoints-corresponding-to-the-face-parts)
    *   [Drei Facemesh Helper (Indices)](https://drei.pmnd.rs/?path=/story/misc-facemesh--face-mesh-standard-story)
*   **THREE.js:**
    *   [CDN Link (cdnjs)](https://cdnjs.com/libraries/three.js/)
    *   [CDN Usage Forum Thread](https://discourse.threejs.org/t/how-to-use-three-js-via-cdn/16836)
*   **Examples:**
    *   [GitHub Demo using FaceMesh JS](https://github.com/google/mediapipe/blob/master/docs/solutions/face_mesh.md#javascript-solution-api)
    *   [Medium Eye/Nose/Mouth Detection Example](https://medium.com/analytics-vidhya/eye-aspect-ratio-ear-and-mouth-aspect-ratio-mar-using-mediapipe-face-mesh-18df8c61534c)

All libraries used are under permissive licenses (MIT / Apache-2.0), making bundling and modification straightforward.

Enjoy the portal!
