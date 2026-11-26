# Raptor — Virtual Pitch-Side Ad Injection Platform (Approved Architecture & Design)

**Competition:** SpiritX 2.0 (Organized by MoraSpirit) — *"Where Sports Meets Technology"*  
**Streaming Architecture:** **Method A — Pre-OBS Smart Virtual Camera Engine**

---

## 1. System Overview & Core Concept

**Raptor** is a live sports broadcast enhancement and micro-sponsorship platform built for university and grassroots sports streams (e.g., MoraSpirit, SLUG, inter-university tournaments, and school matches).

Instead of requiring expensive physical LED perimeter boards or PVC flex banners, Raptor lets a human cameraman film naturally with any mini-camera/DSLR/webcam. Whenever the cameraman points the camera at a **preconfigured angle** on the pitch, Raptor automatically recognizes the view, injects the currently scheduled sponsor's virtual billboard in 3D perspective, blends it with the stadium lighting, keeps players in front of the ad, and outputs the composited feed directly into **OBS Studio** as a Virtual Camera.

---

## 2. Approved Streaming Pipeline (Method A)

Raptor acts as an intelligent pre-OBS camera processor—fitting seamlessly into existing MoraSpirit/university broadcast workflows without replacing OBS.

```text
[Mini Camera / DSLR / Phone / Match Clip]
        │  (USB UVC / HDMI Capture Card / Local RTSP)
        ▼
[Raptor CV & Ad-Injection Engine (Python + OpenCV)]
  1. Preconfigured Angle Recognition (ORB/AKAZE + USAC_MAGSAC)
  2. Smooth 30 FPS Lock (Lucas-Kanade Optical Flow + One-Euro Filter)
  3. 3D/Perspective Billboard Warp (solvePnP + warpPerspective)
  4. Brand-Safe Color & Lighting Assist (Lab Luminance + White Balance)
  5. Player Occlusion (Turf Chroma-Key / ROI Segmentation)
  6. Non-Blocking Timeslot Ad Switcher (2D Crossfade/Slide)
        │
        ├──────────────────► [OBS Virtual Camera / Local RTSP] ──► [OBS Studio] ──► [YouTube / FB Live]
        │                                                           (Scoreboard, Audio, Replay)
        ▼
[Web Preview & Telemetry] ◄── WebSocket / REST ──► [FastAPI Backend] ◄──► [Next.js Web App]
                                                    • Timeslot DB           • Broadcaster Studio
                                                    • Clock Sync            • Local Shop Marketplace
                                                    • Proof-of-Play         • Sponsor Analytics
```

### Why Method A Was Approved
- **Zero Workflow Disruption:** University media crews already use OBS Studio for scoreboards, commentary audio, and scene switching. They simply add **"Raptor Virtual Pitch Cam"** as a Video Capture Source in OBS.
- **Zero-Frame Sync Drift:** Ads are composited directly onto the video frames *before* OBS encodes and streams to YouTube/Facebook Live.
- **No `aiortc` / Browser-Tab Bottlenecks:** Native Python + OpenCV + C++/VirtualCam runs reliably at 30 FPS on a standard laptop.

---

## 3. Core Technical Modules

### A. Preconfigured Camera Angles ("Empty Views")
- **Pre-Match Calibration:** The cameraman points the camera at 2–4 key pitch positions (e.g., *Behind Goalpost*, *Main Sideline*, *Pavilion End*) and captures each as a **Reference Angle**.
- **Visual Fingerprinting:** OpenCV extracts static keypoints (pitch line intersections, goalposts, fences, stands) using **ORB/AKAZE** and stores each angle's 3D/perspective billboard quad `(x, y, z, yaw, tilt, width, height)`.
- **Live Angle Detection & Lock:**
  - A background thread matches live frames against saved Reference Angles (`BFMatcher` + `USAC_MAGSAC`).
  - Once matched ($\ge 25$ geometric inliers), **Lucas-Kanade Optical Flow** tracks the keypoints at 30 FPS with a **One-Euro Filter** to eliminate micro-jitter.
  - When the cameraman pans away, confidence drops and the billboard smoothly fades out (`150 ms` decay) so the rest of the stream stays clean.
  - **Demo Resilience Mode:** Supports both **Natural Keypoint Tracking** (real match footage/outdoor pitch) and **ArUco Marker Mode** (for reliable indoor tabletop demos under harsh auditorium lighting).

### B. 3D Angled Billboard Projection & Surface Angle Detection
- **Surface Angle Detection (CamScanner-Style Auto-Snap):**
  - **Empty Board / Mat Auto-Snap:** When the operator clicks inside an existing blank perimeter board or mat, OpenCV runs `GaussianBlur` → `Canny` → `findContours` → `approxPolyDP` to detect the 4-corner quadrilateral and automatically snap the billboard to that surface's exact perspective angle.
  - **Pitch-Line Magnetic Alignment:** On open grass without a physical board, white-pixel thresholding + `cv2.HoughLinesP` detects touchlines and goal-lines so the operator's billboard magnetically snaps to the pitch's vanishing angle.
- **3D Positioning & Locking Techniques:**
  - **Primary (Tripod PTZ — Fastest Setup):** Operator places the 4 corners (or uses **3D Pitch / Yaw / Roll sliders** for standing $75^\circ\text{–}90^\circ$ boards vs. flat $0^\circ$ decals) on the Reference Angle. Live, `cv2.perspectiveTransform(corners, H)` locks the angled billboard to the camera's pan/tilt/zoom.
  - **Full 3D Pose (`solvePnP`):** For moving cameras, 4 ground-plane points recover $(R, t)$ via `cv2.solvePnPRansac` (`SOLVEPNP_IPPE`), projecting 3D quads via `cv2.projectPoints` + `cv2.warpPerspective`.
  - **Demo Mode (`ArUco`):** 2–4 subtle `cv2.aruco` markers on a tabletop pitch mat provide instant 60 FPS 3D pose locking under harsh indoor lighting.

### C. Brand-Safe Color & Lighting Assist
To prevent the "floating sticker" look without distorting trademarked sponsor colors:
1. **Ambient White-Balance:** Samples nearby white pitch lines to detect warm/cool stadium lighting and applies a clamped 10–20% tint.
2. **Luminance-Only Matching (`CIELAB` $L^*$):** Matches black-lift and highlight compression to the surrounding pitch so `#000000` matches real camera haze.
3. **Shadow & Turf Texture:** Multiplies flat pitch decals by local pitch luminance so shadows fall naturally across the ad.
4. **Optical Polish:** Subtle lens defocus ($\sigma \approx 0.6\text{–}1.0\text{ px}$), directional motion blur during fast pans, and 1–2 px edge feathering.
5. **Brand-Safe Blend Slider:** Lets the operator balance between *Natural Broadcast* and *Brand-Locked Contrast*.

### D. Player Occlusion
- **Flat Pitch Decals:** Fast adaptive **green-turf chroma keying** (Lab/HSV) inside the billboard ROI so players' boots and legs appear *on top* of the ad.
- **Standing Perimeter Boards:** **ROI-cropped `YOLOv8n-seg` (ONNX)** run only on the tight bounding box around the billboard.

### E. Self-Serve Sponsor Marketplace & Real-Time Timeslot Switching
- **Local Shop Self-Serve Dashboard:** Campus-adjacent businesses (eateries, bookshops, tuition centers) select a match, pick a **Camera Angle**, choose a timeslot (e.g., 5-minute blocks; 15–30s slots for live demos), upload or generate a banner, and complete checkout (LKR / mock LankaQR).
- **30-Second Ad Creative Generator:** Generates high-contrast, pitch-legible horizontal banners from 3 inputs (*Shop Name*, *Offer*, *Contact/QR*) for shops without designers.
- **Zero-Blocking Sync Loop:**
  - The 30 FPS CV loop **never** queries the database or network.
  - FastAPI pushes `SCHEDULE_UPDATED` over WebSocket; a background thread pre-downloads and decodes upcoming ads into RAM as `np.ndarray`s.
  - Transitions (600 ms crossfade or slide) are blended in **flat 2D texture space *before* `warpPerspective()`**—guaranteeing zero flicker.
  - Server-synced clock (`GET /api/time`) ensures dashboard countdowns and live billboard flips stay synchronized.

### F. Verified Proof-of-Play & Fair Metering
- **Screen-Time Metering (CPV):** Purchased ad seconds only decrement when the CV engine confirms the billboard is actively visible on screen ($\ge 2\%$ frame area). If the cameraman never points at that angle during a slot, unused time rolls over.
- **Post-Match Sponsor Certificate:** Logs exact seconds aired, average visibility %, and auto-captured frame snapshots into a shareable Proof-of-Play report.

---

## 4. Built-In Risk Mitigations

| Risk | Approved Mitigation |
| :--- | :--- |
| **Tracking Collapse / Whip-Pan** | Confidence gating fades opacity to 0% over 150 ms instead of glitching; `Spacebar` **Emergency Hide Hotkey** instantly clears all virtual ads without interrupting OBS. |
| **Tripod Bump Mid-Match** | **1-Click Smart Recalibrate** re-anchors the reference frame on a paused preview in <10 seconds while keeping the live OBS feed clean. |
| **Inappropriate / Competitor Ads** | Pre-flight resolution/legibility check + **Category Exclusivity Locks** + 1-click Organizer Approval queue (unapproved slots fall back to a safe **House Ad**). |
| **Double-Booking Race Conditions** | Atomic DB reservation locks + 5-minute checkout hold; **Match-Phase Anchoring** with a broadcaster **"Pause Ad Clock"** button for rain/injury delays. |
| **UoM Finals Venue Wi-Fi / Lighting** | 100% local-first LAN execution + **Dual-Input Switcher** (toggle between **Real UoM Match Video** and **Live Mini-Camera / Webcam**). |

---

## 5. Hackathon MVP Scope vs. Future Roadmap

### Phase 2 MVP (Build Now)
- **CV Engine (Method A):** Preconfigured angle capture, ORB/AKAZE + LK tracking (plus ArUco demo mode), 3D/perspective quad warp, Brand-Safe Color Assist slider, turf player occlusion, `Spacebar` Emergency Hide, Smart Recalibrate, and **OBS Virtual Camera** (`pyvirtualcam`) + local Web preview output.
- **Backend (FastAPI + SQLite):** Atomic timeslot booking, WebSocket schedule push, clock sync, local asset cache, and Proof-of-Play logger.
- **Web App (Next.js):** Broadcaster Studio (angle setup + live monitor), Sponsor Marketplace (slot picker + 30s Banner Generator + mock LankaQR checkout), and Proof-of-Play analytics report.

### Future Roadmap (Pitch Deck)
- Direct OBS NDI / Native Plugin packaging.
- Audience-segmented client-side WebGL compositing (geo-targeted ads per viewer region).
- Automated AI content moderation and dynamic Big Match surge auctions.
- Production PayHere / LankaQR gateway integration with automated 70/20/10 revenue splits (70% to MoraSpirit / University Sports Council).

---

## 6. Prototype Build Plan (Step-by-Step)

### Target Repository Structure
```text
raptor-spiritx/
├── cv-engine/                  # Python 3.12 + OpenCV + pyvirtualcam
│   ├── main.py                 # Capture loop (Webcam / Match Video) -> OBS Virtual Cam
│   ├── angle_tracker.py        # ORB/AKAZE Reference Angles + LK Optical Flow + One-Euro
│   ├── surface_snap.py         # CamScanner-style 4-corner snap + Hough line guides
│   ├── compositor.py           # warpPerspective + Brand-Safe Color Assist + Turf Occlusion
│   └── schedule_sync.py        # Non-blocking WS client + pre-decoded RAM ad cache
├── backend/                    # FastAPI + SQLite
│   ├── app.py                  # REST API (/matches, /angles, /slots, /time, /proof)
│   ├── db.py                   # Atomic slot booking + overlap constraints
│   └── ws_hub.py               # Real-time schedule push + Proof-of-Play telemetry
├── web/                        # Next.js (Tailwind)
│   ├── app/studio/             # Broadcaster Studio (Angle capture, 4-corner/3D snap, Hide hotkey)
│   ├── app/marketplace/        # Local Shop Portal (Slot picker, 30s Banner Gen, mock LankaQR)
│   └── app/analytics/          # Proof-of-Play Sponsor Report
└── docs/
    ├── SpiritX.2.0.Delegate.Booklet (1).pdf
    └── ARCHITECTURE_AND_DESIGN.md
```

### 4-Step Build Sequence

1. **Step 1 — Core CV Engine & OBS Virtual Cam Output (`cv-engine/`)**
   - Build camera/video-clip input toggle + `pyvirtualcam` output (verified directly inside OBS Studio as `"Raptor Virtual Pitch Cam"`).
   - Implement **Reference Angle Capture** + **CamScanner 4-Corner Auto-Snap** + manual drag/3D tilt handles.
   - Implement **Angle Recognition (`ORB/AKAZE` + `USAC_MAGSAC`)** + **Lucas-Kanade Optical Flow** + **One-Euro Filter** + `150 ms` fade-out confidence gate (plus `ArUco` Demo Mode toggle).
   - Add **Brand-Safe Color Assist** (white-line balance + `CIELAB` $L^*$ match + blend slider) and **Green-Turf Player Occlusion**.

2. **Step 2 — FastAPI Timeslot Scheduler & Sync (`backend/`)**
   - SQLite schema for `matches`, `angles`, `ad_creatives`, `ad_bookings`, and `proof_of_play_logs` with atomic overlap checks.
   - Server clock endpoint (`GET /api/time`), image upload/auto-resize, and WebSocket `SCHEDULE_UPDATED` broadcast.
   - Connect `schedule_sync.py` so the CV engine pre-loads upcoming ads in RAM and crossfades them in 2D texture space with zero frame drops.

3. **Step 3 — Web Studio, Marketplace & Banner Generator (`web/`)**
   - **Broadcaster Studio:** Live preview, "Save Reference Angle" button, CamScanner/4-corner surface placer, Brand-Safe Blend slider, `Spacebar` Emergency Hide, and Smart Recalibrate.
   - **Sponsor Marketplace:** Visual 15s/30s/5m timeslot picker per Camera Angle, **30-Second Banner Generator** (Shop Name + Offer + Contact/QR → PNG), 1-click Organizer Approval, and mock LankaQR checkout.
   - **Proof-of-Play Dashboard:** Live verified screen-time counter + exportable Sponsor Certificate.

4. **Step 4 — End-to-End Verification (Method A Demo Check)**
   - **Test 1 (OBS Integration):** Open OBS Studio → Add `"Raptor Virtual Pitch Cam"` → verify 30 FPS clean feed.
   - **Test 2 (Angle Lock):** Point webcam at Angle 1 → ad locks in perspective; pan away → ad fades out; pan back → ad re-locks.
   - **Test 3 (Live Timeslot Switch):** Book the next 15s slot in the Marketplace → watch the ad crossfade live on the pitch inside OBS → verify Proof-of-Play seconds increment.
   - **Test 4 (Real Match Footage):** Switch input to a real UoM/sports match clip → verify players run *in front* of the pitch ad and color blending looks natural.
