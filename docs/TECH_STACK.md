# Raptor — Tech Stack & Exploration Options

> **Note for the Team:** All technologies and methods listed below are our **current working recommendations** and **can change** as we prototype and benchmark. Each section includes **multiple alternative approaches** for team members to explore, test, and compare.

---

## 1. Camera Input & Video Ingestion

| Option | How It Works | Pros | Cons / When to Swap |
| :--- | :--- | :--- | :--- |
| **USB / HDMI Capture (`cv2.VideoCapture`)** *(Recommended)* | Reads mini-camera, webcam, or DSLR (via HDMI-to-USB capture card) directly in Python. | Simple, universal, low latency. | Dependent on USB bandwidth for 1080p60. |
| **Wireless Phone Camera (RTSP / NDI / DroidCam)** | Uses a smartphone on the pitch streaming over local Wi-Fi/hotspot. | Lets the cameraman move freely around the ground. | Adds 100–300 ms Wi-Fi latency; vulnerable to interference. |
| **Pre-Recorded Match Clip (`.mp4`)** *(Demo Fallback)* | Feeds a real university match video through the exact same pipeline. | 100% repeatable for testing occlusion & lighting; zero venue risk. | Not live—use as a toggle alongside the live camera. |

---

## 2. Angle Recognition & Camera Tracking

| Option | How It Works | Pros | Cons / Exploration Notes |
| :--- | :--- | :--- | :--- |
| **`ORB / AKAZE` + `USAC_MAGSAC` + `Lucas-Kanade` + `One-Euro Filter`** *(Recommended)* | Background thread matches live frames against saved Reference Angles; LK Optical Flow locks at 30 FPS; One-Euro smooths jitter. | Runs at 30 FPS on CPU; handles slight pan/tilt/zoom; zero extra hardware. | Struggles on 100% plain grass with no lines/landmarks. *(Explore `AKAZE` vs `ORB` for zoom tolerance.)* |
| **`SIFT` (OpenCV)** | Classic scale-invariant feature detector. | More accurate across heavy zoom changes than ORB. | Slower on CPU (~2–3×) than ORB; test if 30 FPS holds. |
| **Deep Feature Matching (`SuperPoint` + `LightGlue`)** | Neural network keypoint extractor & matcher (ONNX/PyTorch). | Dramatically better under harsh lighting/blur. | Requires GPU/ONNX Runtime; heavier setup for MVP. |
| **`ArUco` / `AprilTag` Markers** *(Demo Mode)* | Detects small printed fiducial markers placed on a mini-pitch mat. | Instant 60 FPS lock, immune to bad auditorium lighting. | Requires physical markers—best for tabletop/booth demos. |
| **Hardware Encoders (`FreeD` Protocol)** | Physical rotary sensors on tripod head + lens. | Industry-standard sub-pixel accuracy even on plain grass. | Expensive hardware; out of scope for software-only MVP. |

---

## 3. 3D Billboard Positioning & Surface Angle Detection

| Option | How It Works | Pros | Cons / Exploration Notes |
| :--- | :--- | :--- | :--- |
| **Direct 4-Corner / 3D Tilt Gizmo + Homography Transfer** *(Recommended for Tripod PTZ)* | Operator places 4 corners (or tilts with Pitch/Yaw/Roll sliders) on Reference Angle; live frames transform corners via `cv2.perspectiveTransform`. | Zero 3D calibration needed; works for both flat decals and angled standing boards on a tripod. | Assumes minimal camera walking (rotation/zoom only). |
| **3D Ground Plane (`solvePnPRansac`) + `projectPoints`** | Operator marks 4 pitch points to recover $(R, t)$, then places 3D quads in $(X, Y, Z)$. | Handles handheld/moving cameras with true 3D parallax. | Can wobble if focal length $K$ isn't calibrated. |
| **CamScanner-Style Auto-Snap (`Canny` + `findContours` + `approxPolyDP`)** | Automatically detects 4-sided blank boards/mats near a user click and snaps corners to their angle. | Fast operator UX on existing physical boards. | Needs clear edges around the board. |
| **Pitch-Line Magnetic Snap (`cv2.HoughLinesP`)** | Thresholds white pitch lines and snaps billboard edges to touchline/goal-line perspective. | Works on open grass without a physical board. | Sensitive to faded grass lines or harsh shadows. |
| **WebGL / `Three.js` 3D Renderer** | Sends $(R, t)$ or homography to a WebGL shader to render `PlaneGeometry` meshes. | Hardware mipmapping (no aliasing at steep angles) + `<1 ms` shaders. | Harder to integrate back into Python → OBS Virtual Cam. |

---

## 4. Brand-Safe Color & Lighting Assist

| Option | How It Works | Pros | Cons / Exploration Notes |
| :--- | :--- | :--- | :--- |
| **White-Line Balance + `CIELAB` $L^*$ Luminance Match + Blend Slider** *(Recommended)* | Samples white pitch lines for ambient temp (clamped 10–20% tint) and matches black/white levels in $L^*$ channel only. | Preserves trademarked brand colors while matching shadows & haze. | Requires nearby white/neutral pixels or fallback ROI stats. |
| **HSV Value-Only Matching** | Scales only the `V` (brightness) channel to match local pitch luminance. | Simpler and faster than Lab conversion. | Doesn't adapt to warm sunset vs. cool floodlight color temps. |
| **GLSL Fragment Shader Compositing** | Performs color transfer, shadow multiply, and edge feathering on GPU. | Sub-millisecond execution at 1080p/4K. | Only applies if rendering in OpenGL/WebGL. |

---

## 5. Player Occlusion (Keeping Players in Front of Ads)

| Option | How It Works | Pros | Cons / Exploration Notes |
| :--- | :--- | :--- | :--- |
| **Adaptive Green-Turf Chroma Key (Lab / HSV)** *(Recommended for Flat Pitch Ads)* | Auto-clusters grass color inside the billboard ROI; draws ad only over turf pixels. | ~1 ms on CPU; pixel-accurate around boots/cleats. | Fails if a player wears a bright green jersey matching the turf. |
| **ROI-Cropped `YOLOv8n-seg` / `YOLO11n-seg` (ONNX)** *(Recommended for Standing Boards)* | Runs person segmentation *only* inside the billboard's bounding box. | Works on any background (crowd, fences, green kits). | ~5–8 ms on GPU, ~15 ms on CPU; test ONNX/OpenVINO speed. |
| **MediaPipe Selfie / Image Segmentation** | Lightweight mobile-first foreground segmentation. | Very fast in browser/WASM or CPU. | Struggles on small, distant players in wide stadium shots. |

---

## 6. Local Streaming & OBS Integration (Method A)

| Option | How It Works | Pros | Cons / Exploration Notes |
| :--- | :--- | :--- | :--- |
| **`pyvirtualcam` → OBS Virtual Camera** *(Recommended)* | Python pushes composited frames directly as a system Virtual Camera (`"Raptor Virtual Pitch Cam"`) into OBS Studio. | Zero extra network hop; plugs straight into OBS like a normal webcam. | Requires OBS Virtual Cam driver (or UnityCapture) installed on Windows. |
| **FFmpeg → Local `MediaMTX` (RTSP) → OBS Media Source** | Python pipes raw frames to FFmpeg → local RTSP/RTMP server → OBS reads the local stream. | Decouples CV process from OBS; also serves WebRTC/LL-HLS to a web viewer. | Adds ~200–500 ms buffer latency; must drain FFmpeg `stderr`. |
| **NDI Output (`ndi-python`)** | Sends uncompressed/low-latency video over local network via NDI to OBS. | Broadcast-standard, supports alpha channel & multi-machine setups. | Requires NDI Runtime + `obs-ndi` plugin installed. |

---

## 7. Backend, Scheduler & Real-Time Sync

| Option | How It Works | Pros | Cons / Exploration Notes |
| :--- | :--- | :--- | :--- |
| **`FastAPI` (Python) + `SQLite` + `WebSockets`** *(Recommended)* | Single-process API handling slot bookings, clock sync (`/api/time`), Proof-of-Play logs, and pushing schedule updates to CV engine. | Same language as CV engine; zero-config SQLite; fast to build. | Single-writer limit in SQLite (mitigated via `BEGIN IMMEDIATE`). |
| **`PostgreSQL` (with Exclusion Constraints)** | Replaces SQLite for production-grade overlapping-timeslot prevention. | Native `EXCLUDE USING gist` prevents double-booking at DB level. | Slightly heavier local setup than SQLite. |
| **`Node.js` (`Express`/`NestJS`) + `Redis Pub/Sub`** | Dedicated JS/TS backend with Redis for multi-worker schedule fan-out. | Great if frontend team prefers full-stack TypeScript. | Extra service to run during hackathon demo. |

---

## 8. Web Frontend (Studio, Marketplace & Analytics)

| Option | How It Works | Pros | Cons / Exploration Notes |
| :--- | :--- | :--- | :--- |
| **`Next.js` (React) + `Tailwind CSS` + `HTML5 Canvas`** *(Recommended)* | Web app for Broadcaster Studio, Self-Serve Shop Marketplace, and Proof-of-Play Dashboard. | Fast UI iteration, responsive for mobile shop owners, easy Canvas corner editor. | Needs local backend running for live camera preview. |
| **`Vite` + `React`** | Pure client-side SPA without Next.js SSR overhead. | Faster dev server startup; simpler static hosting. | No built-in API routes if needed. |
| **Desktop Operator UI (`PyQt6` / `OpenCV HighGUI` / `Tauri`)** | Native desktop window for the cameraman/operator alongside the web marketplace. | Zero browser lag for 60 FPS corner calibration & hotkeys (`Spacebar` hide). | Harder to style cleanly than Tailwind/React. |

---

## 9. Banner Generator & Payments

| Option | How It Works | Pros | Cons / Exploration Notes |
| :--- | :--- | :--- | :--- |
| **Template-Based Banner Generator (`Pillow` or `HTML5 Canvas`)** *(Recommended MVP)* | Takes Shop Name, Offer, Contact/QR + brand color → outputs high-contrast horizontal PNG. | Instant (<1s), 100% legible at pitch distance, supports EN/SI/TA fonts. | Fixed layouts (3–5 templates). |
| **AI Image Generation / Background Removal** | Uses an image API to stylize shop banners. | Richer creative visuals for shops. | Risk of unreadable/hallucinated text; slower & needs internet. |
| **Mock `LankaQR` / `PayHere` Sandbox** *(MVP)* | Simulated LKR checkout webhook that immediately confirms the slot. | Zero payment gateway approval wait; reliable offline demo. | Swap for real PayHere/LankaQR merchant API in production. |

---

## Suggested Exploration Tasks for Team Members
- **CV Member:** Benchmark `ORB` vs. `AKAZE` vs. `SIFT` on a 30-second pan/zoom match clip; test `HSV` turf chroma-key vs. `YOLOv8n-seg` ONNX speed on your laptop.
- **Streaming Member:** Test `pyvirtualcam` into OBS Studio on Windows vs. `FFmpeg → MediaMTX` RTSP into OBS.
- **Backend/Frontend Members:** Prototype the `HTML5 Canvas` 4-corner + CamScanner snap editor and the 30-second Shop Banner Generator.
