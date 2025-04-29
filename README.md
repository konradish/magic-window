# Face-Tracked 3D Window

**tl;dr** – This page spins up MediaPipe FaceMesh to grab eye landmarks from your webcam, pipes them into a tiny math shim, and offsets a THREE.js perspective-camera so the cube in front of you parallax-shifts as your head moves. Open the file over HTTPS, allow camera access, and your monitor turns into a faux “window” on a 3-D scene. Zero external servers are involved; everything runs client-side.

## How it works

1.  **Webcam + Face Tracking**
    *   [MediaPipe FaceMesh](https://github.com/google/mediapipe/blob/master/mediapipe/modules/face_geometry/README.md) runs in WebAssembly and streams 468 facial-landmark points in real-time (≈15 ms on a mid-tier laptop). ([jsDelivr CDN](https://www.jsdelivr.com/package/npm/@mediapipe/face_mesh))
    *   The helper [@mediapipe/camera\_utils](https://www.npmjs.com/package/@mediapipe/camera_utils) ([GitHub](https://github.com/google/mediapipe/tree/master/mediapipe/web/solutions/camera_utils)) automagically handles `getUserMedia`, pumps frames into the model, and calls back every render tick.
    *   Landmark indices `159` (left-eye lower-mid) and `386` (right-eye lower-mid) are used as a stable “gaze barycenter”. ([Reference 1](https://www.selvam.tech/blogs/mediapipe-facemesh-keypoints/), [Reference 2](https://stackoverflow.com/questions/66979377/mediapipe-face-mesh-keypoints-corresponding-to-the-face-parts))

2.  **Mapping Head Pose → Camera Offset**
    *   Landmarks arrive in normalized screen coordinates (0-1).
    *   Subtracting 0.5 centers the coordinates.
    *   Multiplying by a `MAX_OFFSET` (e.g., 2 world-units) scales the movement.
    *   The result feeds into a simple low-pass filter (`targetX += (newX - targetX) * SMOOTHING`) so the camera eases instead of jittering.
    *   X-movement pans the camera horizontally; Y-movement pans vertically (inverted so moving your head up moves the camera up).
    *   The camera always uses `.lookAt(scene.position)` so the cube stays centered while its parallax effect sells the depth.

3.  **Rendering the Window**
    *   [THREE.js](https://threejs.org/) (via [cdnjs CDN](https://cdnjs.com/libraries/three.js/)) draws a simple lit, rotating cube. This can be replaced with more complex scenes, GLTF models, etc.
    *   The canvas is placed within a flexbox `div`. Resize listeners keep the aspect ratio and projection matrix updated.

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

4.  **Calibrate Depth:**
    *   You'll see buttons "Set Near Point" and "Set Far Point" at the bottom.
    *   **Lean fully towards your screen** to the closest comfortable distance you want tracked. Click **"Set Near Point"**.
    *   **Lean back** to the furthest comfortable distance you want tracked. Click **"Set Far Point"**.
    *   The status text will update. If it shows "Calibrated", the depth tracking is active. If it shows an error (e.g., Near <= Far), try calibrating again.
    *   *Note:* Calibration uses the distance between your eyes in the video feed. Ensure your face is clearly visible during calibration clicks.

5.  **Move your head:**
    *   Move side-to-side and up/down: The scene shifts like looking through a window.
    *   Lean in/out (after calibration): The camera should zoom in/out, enhancing the 3D effect.

## Extensibility Ideas

| Tweak                 | What to change                                                                                                |
| :-------------------- | :------------------------------------------------------------------------------------------------------------ |
| Use nose-tip + depth  | Sample landmark `1` (nose tip) and potentially `199` (forehead)? Use the `z` coordinate to map to `camera.position.z`. |
| Add iris tracking     | Use `refineLandmarks: true` (already enabled) and indices `468`-`477` for pupils. ([Ref](https://github.com/tensorflow/tfjs-models/blob/master/face-landmarks-detection/mesh_map.jpg)) |
| Improve stability     | Swap simple exponential smoothing for a [One-Euro Filter](https://cristal.univ-lille.fr/~casiez/1euro/) or Kalman filter. |
| Richer scene          | Load a GLTF model using `THREE.GLTFLoader`; add post-processing effects.                                      |
| Privacy toggle        | Add a UI switch that calls `cameraUtils.stop()` / `cameraUtils.start()` to pause/resume the camera stream.      |

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
