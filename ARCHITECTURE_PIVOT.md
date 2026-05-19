# Recordly Architecture Pivot: Electron to Tauri (Rust)

## 1. The Core Problem (Maxay u shaqayn la'dahay hadda?)
Recordly is currently built on **Electron** (Node.js + Chromium). While this works reasonably well for macOS (due to Apple's ScreenCaptureKit integration), it catastrophically fails on **Linux (Wayland/Hyprland)** and struggles with performance and accuracy on **Windows**.

**Why AI (and developers) fail to fix it in Electron:**
*   **Wayland Security Restrictions:** Wayland deliberately isolates applications. Chromium (which powers Electron's `desktopCapturer` and WebRTC) uses generic xdg-desktop-portals. These portals **do not** provide granular control over cursor coordinates or independent window tracking required for a "Screen Studio" style smooth-zoom/compositing effect.
*   **Hyprland (Tiling WM) Context:** Hyprland relies on specific IPC sockets and `wlr-protocols`. Electron has no native bindings to communicate directly with these low-level sockets.
*   **Windows Latency:** Connecting Electron's IPC to native C++ (WGC - Windows Graphics Capture) creates latency, resulting in "double cursors" and offset bugs.

## 2. The Solution (Xalka & Hadafka)
To build a true "Screen Studio clone" for Linux and Windows, we must bypass the browser engine (Chromium) for hardware-level tasks. 

**The Stack:** **Tauri (Rust + React)**
*   **Frontend:** We keep the existing React/Vite/Tailwind UI. It's beautiful and works perfectly.
*   **Backend:** We replace Node.js/Electron with **Rust**.
*   **Capture Engine:** We write native Rust modules that directly interface with the OS, bypassing any browser portals.

## 3. Architecture Design (Qaab-dhismeedka Cusub)

### A. Frontend (UI Layer - React)
*   **What stays:** All React components, Tailwind CSS styling, Zustand/Context state management.
*   **What changes:** The IPC (Inter-Process Communication) layer. Instead of `window.electron.ipcRenderer.invoke`, we will use `@tauri-apps/api/invoke`.

### B. Core Backend (Rust Engine)
Instead of running a heavy Node.js background process, a lightweight Rust binary will manage the application lifecycle and native OS interactions.

*   **Linux/Wayland Module:**
    *   **Video Capture:** Rust using `pipewire-rs` and `ashpd` to request raw screen/window buffers directly from the compositor.
    *   **Cursor Tracking:** Rust connecting directly to the Hyprland IPC socket (or using `wlr-protocols`) to poll X/Y coordinates in real-time.
*   **Windows Module:**
    *   **Video Capture:** Rust using `windows-rs` to implement `Windows.Graphics.Capture` (WGC) natively, zero IPC overhead between the capture engine and the backend.
    *   **Cursor Tracking:** Native Win32 API (`GetCursorInfo`) polled within a high-performance Rust thread.

## 4. Step-by-Step Transition Plan (Tallaabooyinka Shaqada)

### Step 1: Environment & UI Extraction
*   **Action:** Initialize a new Tauri project (`npm create tauri-app`). Copy the `src`, `public`, and `index.html` from the current Recordly project into the new Tauri structure.
*   **Why:** We must isolate the UI from Electron dependencies before writing any backend code.
*   **Expectation:** The UI will load in a Tauri window, but buttons (like "Start Recording") will do nothing because the backend is missing.

### Step 2: Rust Recording Engine (Proof of Concept)
*   **Action:** Write a Rust module (`src-tauri/src/capture`) that can record a dummy 5-second video.
    *   *Linux:* Implement PipeWire connection.
    *   *Windows:* Implement WGC connection.
*   **Why:** To prove that Rust can capture the screen flawlessly on Wayland and Windows without Chromium getting in the way.
*   **Expectation:** A native, hardware-accelerated video file is saved to the disk.

### Step 3: Independent Cursor Tracking
*   **Action:** Write a Rust background thread that continuously logs the exact X, Y coordinates of the mouse cursor, completely separate from the video stream.
    *   *Linux:* Read from Hyprland IPC.
*   **Why:** Screen Studio's magic relies on drawing the cursor *after* the recording is done. We need a perfect timestamped JSON log of where the cursor was.
*   **Expectation:** A highly accurate JSON file mapping timestamps to screen coordinates.

### Step 4: Compositing (The Magic)
*   **Action:** Port the "Video + Cursor Overlay" logic to Rust. Use a Rust media library (like `ffmpeg-next`) to take the raw video (from Step 2) and draw an SVG cursor on top of it using the coordinates (from Step 3).
*   **Why:** Doing this in Rust is 100x faster than doing it in Node.js or the browser.
*   **Expectation:** The final exported video has a buttery smooth, perfectly aligned cursor.

## Conclusion
Patching the current Electron architecture is a dead end for Linux Wayland/Hyprland. By migrating to Tauri, we retain the beautiful UI but gain the low-level system access required to make a professional, high-performance screen recorder. We build exactly what the OS expects, not what Chromium allows.