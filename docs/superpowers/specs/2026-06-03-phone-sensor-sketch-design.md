# Phone Sensor Sketch — Design

Date: 2026-06-03
Status: Approved for planning

## Goal

A new, self-contained variant of the p5 camera-shape-collage sketch built to run
on an Android phone with **no touch input during the run** (a single start tap for
permissions/setup is allowed). Phone sensors — microphone, device tilt, camera
pixels, accelerometer — generatively control when, where, and how big the rotating
shape cutouts are drawn. The core visual effect is unchanged from `index.html`.

## Non-Goals

- No ml5 / FaceMesh / blink detection (audio-peak triggering was chosen instead).
- No added color, tint, outline, or glow. The shape interior remains the live
  video cutout exactly as in `index.html`; that live feed *is* the camera-driven
  color.
- No toolbar, no drag-to-draw, no per-shape manual control.
- No change to the rendering core (cover-fit video + rotating clip-and-snapshot
  cutout).

## File

`index-sensors.html` — a single self-contained HTML file, forked from `index.html`
(the simpler, no-ml5 sketch). p5.js 2.x via the same CDN tag. The rendering
functions (`videoRect`, `drawVideoCover`, `makeShape`, `shapeCenter`, `tracePath`,
`drawShapeCutout`) are copied unchanged. The drag/toolbar/keyboard input code is
removed and replaced by the sensor engine below.

## User Flow

1. **Start screen.** Black screen showing "Tap to start" and a small Front / Rear
   camera selector (default Front). Nothing runs yet.
2. **The one tap.** On tap, the sketch:
   - requests camera (chosen facing) + microphone via `getUserMedia`,
   - creates/resumes the Web Audio `AudioContext` (browsers require a user gesture),
   - begins listening to `deviceorientation` and `devicemotion`, calling
     `DeviceOrientationEvent.requestPermission()` / `DeviceMotionEvent.requestPermission()`
     first if those exist (iOS path; harmless no-op on Android Chrome),
   - captures the current device orientation as the **neutral tilt center**.
3. **Hands-free run.** The generative engine spawns shapes from sensor input. No
   further touch is needed. A hard shake clears the canvas.

## Generative Engine

### When — audio onset trigger
- A Web Audio `AnalyserNode` reads the mic. Each frame compute short-term loudness
  (RMS of the time-domain buffer).
- Maintain an adaptive baseline (slow-moving average of loudness).
- Fire a spawn when current loudness exceeds `baseline * RISE_FACTOR` (and above a
  floor, to ignore silence noise) **and** at least `COOLDOWN_MS` (~250 ms) have
  passed since the last spawn. One clap/word = one shape.

### How (size) — loudness
- The loudness value at the moment of the spike maps to shape size:
  `size = map(loudness, QUIET, LOUD, MIN_SIZE, MAX_SIZE)` clamped. Louder = bigger.
- Width and height derive from `size` with a small random aspect jitter so shapes
  aren't all square-proportioned.

### Where — tilt cursor blended with camera hotspot
- **Tilt cursor:** from `deviceorientation`, take `gamma` (left/right) and `beta`
  (front/back) relative to the captured neutral center, smooth them (low-pass), and
  map to a screen position. Clamp to canvas with margin.
- **Camera hotspot:** every ~10 frames, draw the current video into a tiny
  offscreen buffer (e.g. 32×18), scan cells, and pick the brightest / most-saturated
  cell. Map its cell coords to screen space (accounting for the selfie mirror when
  the front camera is used).
- **Blend:** final spawn position = `lerp(tiltCursor, hotspot, BLEND)` with
  `BLEND ≈ 0.5`. Handheld → tilt leads; propped still → hotspot leads.

### Spin — unchanged
- Each shape gets a constant random spin speed and direction at creation, exactly
  as `makeShape` does today. No sensor affects spin.

### Type
- Randomly cycled among `square`, `circle`, `triangle` per spawn.

### Haptic feedback (optional, on by default)
- `navigator.vibrate(15)` on each spawn where supported. Output only; not a sensor.

## Lifespan & Reset

- **Strictly infinite buildup.** Shapes accumulate forever, like the current sketch.
- **Shake-to-clear:** `devicemotion` acceleration magnitude is monitored. When a
  hard jolt exceeds `SHAKE_THRESHOLD` (with its own cooldown to avoid repeats), the
  shapes array is emptied — a hands-free reset valve. Optional short double-buzz
  haptic to confirm.

## Permissions & Secure Context

- Served over HTTPS via an ngrok tunnel to a local static server
  (e.g. `python3 -m http.server 8000` then `ngrok http 8000`, open the https URL on
  the phone). Camera, mic, and motion sensors require a secure context; `file://`
  will not grant them.
- Android Chrome grants `deviceorientation`/`devicemotion` without an explicit
  prompt over HTTPS; the iOS `requestPermission()` guards are included for
  robustness and are no-ops on Android.

## Performance Notes

- Each shape re-snapshots the entire canvas every frame (this is what produces the
  recursive nesting). With infinite buildup this is O(n) full-canvas copies per
  frame and **will** slow down on a phone after enough shapes. This is an accepted
  tradeoff per the chosen design; shake-to-clear is the mitigation.
- Keep `pixelDensity(1)` for both the main canvas and the snapshot buffer.
- Run the camera-hotspot scan on a throttled cadence (~every 10 frames) on a tiny
  buffer to keep per-frame cost low.

## Tunable Constants (single block near top of file)

`COOLDOWN_MS`, `RISE_FACTOR`, loudness `QUIET`/`LOUD`, `MIN_SIZE`/`MAX_SIZE`,
tilt smoothing factor and sensitivity, `BLEND`, hotspot grid size and scan cadence,
`SHAKE_THRESHOLD`, vibrate durations. Grouped so behavior can be tuned without
hunting through the code.

## Open Risks

- Audio onset thresholds and tilt sensitivity will need on-device tuning; the
  constants block exists for exactly this.
- Camera-hotspot saturation scan may be noisy in low light; brightness fallback if
  saturation is uniformly low.
