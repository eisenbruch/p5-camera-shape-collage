# Phone Sensor Sketch Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build `index-sensors.html`, a hands-free phone variant of the camera-shape-collage sketch where microphone, device tilt, camera pixels, and the accelerometer generatively drive the rotating video cutouts — no touch during the run.

**Architecture:** Fork the simpler `index.html` (no ml5). The rendering core (cover-fit video + rotating clip-and-snapshot cutout) is copied verbatim. Replace the toolbar/drag/keyboard input with: a start-screen state machine (one tap grants camera+mic+motion permissions), a Web Audio onset detector that triggers spawns, a tilt+camera-hotspot position picker, loudness→size mapping, and a shake-to-clear handler. Shapes accumulate infinitely.

**Tech Stack:** p5.js 2.x (CDN), Web Audio API (AnalyserNode), DeviceOrientation/DeviceMotion events, getUserMedia, navigator.vibrate. Verification via Playwright MCP on desktop (synthetic motion events + a debug force-spawn key) and final confirmation on an Android phone over an ngrok HTTPS tunnel.

**Spec:** `docs/superpowers/specs/2026-06-03-phone-sensor-sketch-design.md`

---

## File Structure

- `index-sensors.html` — the entire sketch, single self-contained file. Sections, top to bottom:
  - Tunable constants block (`CFG`)
  - State machine (`appState`: `'idle' | 'running'`) + start screen
  - Rendering core (copied unchanged from `index.html`)
  - Sensor modules: audio onset, orientation/tilt, motion/shake, camera hotspot
  - Spawn logic (combines sensors into a `makeShape` call)

No other files are created or modified. There is no test runner in this repo; verification is browser-based.

---

## Task 1: Scaffold — start screen, render core, run state

**Files:**
- Create: `index-sensors.html`

- [ ] **Step 1: Create the full scaffold file**

Create `index-sensors.html` with this exact content:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no" />
  <title>Sensor Kit — Phone Generative Cutouts</title>
  <script src="https://cdn.jsdelivr.net/npm/p5@2/lib/p5.min.js"></script>
  <style>
    html, body { margin: 0; padding: 0; height: 100%; background: #000; overflow: hidden; }
    canvas { display: block; }
  </style>
</head>
<body>
<script>
/*
  Sensor Kit — a hands-free phone variant of the camera-shape-collage sketch.
  After one start tap (for camera/mic/motion permissions) shapes spawn on their
  own: a mic onset spawns a rotating video cutout, its loudness sets the size,
  device tilt blended with the brightest camera region sets the position, spin is
  constant random per shape (unchanged from index.html), and a hard shake clears
  the canvas. Shapes accumulate forever.
*/

/* ---------------- tunable config ---------------- */
const CFG = {
  cooldownMs: 250,      // min gap between audio-triggered spawns
  riseFactor: 1.8,      // loudness must exceed baseline * this to fire
  loudFloor: 0.02,      // ignore loudness below this (silence noise)
  quiet: 0.03, loud: 0.30,        // loudness range mapped to size
  minSize: 60, maxSize: 360,      // shape size range (px)
  tiltSmooth: 0.15,     // low-pass factor for tilt cursor (0..1, higher = snappier)
  tiltRange: 35,        // degrees of tilt that span half the screen
  blend: 0.5,           // 0 = pure tilt cursor, 1 = pure camera hotspot
  hotspotEveryN: 10,    // frames between camera-hotspot scans
  hotGridW: 32, hotGridH: 18,     // hotspot downsample grid
  shakeThreshold: 28,   // accel magnitude (m/s^2) to trigger clear
  shakeCooldownMs: 1200,
  vibrateSpawn: 15, vibrateClear: [40, 60, 40]
};

/* ---------------- state ---------------- */
let appState = 'idle';     // 'idle' (start screen) | 'running'
let facing = 'user';       // 'user' (front) | 'environment' (rear)
let capture = null;
let snap = null;           // offscreen snapshot of the composite-so-far
let shapes = [];
let startBtn = null, frontBtn = null, rearBtn = null;

function setup() {
  createCanvas(windowWidth, windowHeight);
  pixelDensity(1);
  snap = createGraphics(windowWidth, windowHeight);
  snap.pixelDensity(1);
  angleMode(RADIANS);
  noStroke();
}

function windowResized() {
  resizeCanvas(windowWidth, windowHeight);
  snap.resizeCanvas(windowWidth, windowHeight);
}

function draw() {
  background(0);
  if (appState === 'idle') { drawStartScreen(); return; }
  if (!videoReady()) { drawWaiting(); return; }

  drawVideoCover();
  for (const s of shapes) { s.angle += s.speed; drawShapeCutout(s); }
  drawHud();
}

/* ---------------- start screen ---------------- */

function drawStartScreen() {
  textAlign(CENTER, CENTER);
  fill(255); textSize(min(width, height) * 0.07);
  text('Tap to start', width / 2, height * 0.4);

  // Front / Rear selector
  const bw = min(width * 0.34, 200), bh = 56, gap = 16;
  const cy = height * 0.58;
  frontBtn = { x: width / 2 - bw - gap / 2, y: cy, w: bw, h: bh };
  rearBtn  = { x: width / 2 + gap / 2,      y: cy, w: bw, h: bh };
  startBtn = { x: 0, y: 0, w: width, h: height }; // whole screen starts it

  drawToggle(frontBtn, 'Front', facing === 'user');
  drawToggle(rearBtn, 'Rear', facing === 'environment');

  fill(255, 150); textSize(13);
  text('camera + mic + motion permission needed', width / 2, height * 0.7);
}

function drawToggle(b, label, on) {
  push();
  stroke(255, on ? 255 : 90); strokeWeight(1.5);
  fill(on ? color(255, 235) : color(0, 0, 0, 120));
  rect(b.x, b.y, b.w, b.h, 6);
  noStroke(); fill(on ? 0 : 255); textSize(16);
  text(label, b.x + b.w / 2, b.y + b.h / 2);
  pop();
}

function hit(b, x, y) { return b && x >= b.x && x <= b.x + b.w && y >= b.y && y <= b.y + b.h; }

function handleStartTap(x, y) {
  if (hit(frontBtn, x, y)) { facing = 'user'; return; }
  if (hit(rearBtn, x, y))  { facing = 'environment'; return; }
  beginRun();
}

function beginRun() {
  appState = 'running';
  capture = createCapture({ video: { facingMode: facing }, audio: false }, () => {});
  capture.hide();
  // Audio, orientation, motion permissions/start are wired in later tasks.
}

/* ---------------- input (start tap only) ---------------- */

function mousePressed() {
  if (appState === 'idle') handleStartTap(mouseX, mouseY);
  return false;
}
function touchStarted() { return mousePressed(); }

// Debug: 's' force-spawns a centered shape for desktop testing without sensors.
function keyPressed() {
  if (key === 's' || key === 'S') spawnShape(width / 2, height / 2, 160, 160, 'circle');
}

/* ---------------- rendering core (copied unchanged from index.html) ---------------- */

function videoReady() { return capture && capture.width > 0 && capture.height > 0; }

function videoRect() {
  const vw = capture.width, vh = capture.height;
  const s = Math.max(width / vw, height / vh);
  const dw = vw * s, dh = vh * s;
  return { dx: (width - dw) / 2, dy: (height - dh) / 2, dw, dh };
}

function drawVideoCover() {
  const r = videoRect();
  push();
  if (facing === 'user') { translate(width, 0); scale(-1, 1); } // selfie mirror
  image(capture, r.dx, r.dy, r.dw, r.dh);
  pop();
}

function makeShape(type, x, y, w, h) {
  const bx = Math.min(x, x + w), by = Math.min(y, y + h);
  const bw = Math.abs(w), bh = Math.abs(h);
  const dir = Math.random() < 0.5 ? -1 : 1;
  return { type, x: bx, y: by, w: bw, h: bh, angle: 0,
           speed: dir * (0.004 + Math.random() * 0.016) };
}

function shapeCenter(s) {
  if (s.type === 'triangle') return { cx: s.x + s.w / 2, cy: s.y + (2 * s.h) / 3 };
  return { cx: s.x + s.w / 2, cy: s.y + s.h / 2 };
}

function tracePath(ctx, s) {
  ctx.beginPath();
  if (s.type === 'square') {
    ctx.rect(s.x, s.y, s.w, s.h);
  } else if (s.type === 'circle') {
    ctx.ellipse(s.x + s.w / 2, s.y + s.h / 2, s.w / 2, s.h / 2, 0, 0, Math.PI * 2);
  } else {
    ctx.moveTo(s.x + s.w / 2, s.y);
    ctx.lineTo(s.x + s.w, s.y + s.h);
    ctx.lineTo(s.x, s.y + s.h);
    ctx.closePath();
  }
}

function drawShapeCutout(s) {
  const { cx, cy } = shapeCenter(s);
  snap.clear();
  snap.drawingContext.drawImage(drawingContext.canvas, 0, 0, width, height);
  push();
  translate(cx, cy); rotate(s.angle); translate(-cx, -cy);
  tracePath(drawingContext, s);
  drawingContext.clip();
  image(snap, 0, 0, width, height);
  pop();
}

/* ---------------- spawn (sensor wiring added in later tasks) ---------------- */

const TYPES = ['square', 'circle', 'triangle'];
function spawnShape(x, y, w, h, type) {
  const t = type || TYPES[Math.floor(Math.random() * TYPES.length)];
  shapes.push(makeShape(t, x - w / 2, y - h / 2, w, h));
  if (navigator.vibrate) navigator.vibrate(CFG.vibrateSpawn);
}

/* ---------------- hud ---------------- */

function drawHud() {
  fill(255, 150); textSize(13); textAlign(LEFT, BOTTOM);
  text('shapes: ' + shapes.length, 14, height - 12);
}

function drawWaiting() {
  fill(255); textSize(16); textAlign(CENTER, CENTER);
  text('Allow camera access…', width / 2, height / 2);
}
</script>
</body>
</html>
```

- [ ] **Step 2: Start a local server**

Run: `python3 -m http.server 8000` (from the project root, in a background shell).
Expected: serves the directory; `index-sensors.html` reachable at `http://localhost:8000/index-sensors.html`.

- [ ] **Step 3: Verify the start screen renders (Playwright MCP)**

Navigate to `http://localhost:8000/index-sensors.html`, take a snapshot/screenshot.
Expected: black screen with "Tap to start", Front (highlighted) and Rear toggles. No JS console errors.

- [ ] **Step 4: Verify run state + render core via debug spawn**

In the browser, click the Front toggle, then click center to start (camera may be denied in headless Playwright — that is fine; the goal is no crash). Press `s` a few times.
Run (Playwright evaluate): `document.querySelector('canvas') !== null`
Expected: canvas present, no console errors, `shapes` array grows on `s` presses (check via `window.shapes?.length` is not exposed — instead confirm HUD "shapes:" count rises in screenshot if camera available; if camera denied it shows "Allow camera access…", which is acceptable for this task).

- [ ] **Step 5: Commit**

```bash
git add index-sensors.html docs/superpowers/plans/2026-06-03-phone-sensor-sketch.md
git commit -m "feat: scaffold sensor sketch — start screen, render core, run state"
```

---

## Task 2: Audio onset trigger + loudness-driven size

**Files:**
- Modify: `index-sensors.html` (add audio module, wire into spawn)

- [ ] **Step 1: Add the audio module**

Insert this block before the `/* ---------------- hud ---------------- */` section:

```javascript
/* ---------------- audio onset ---------------- */
let audioCtx = null, analyser = null, audioBuf = null;
let loudBaseline = 0;          // slow-moving average of loudness
let lastSpawnMs = -1e9;

async function startAudio() {
  try {
    const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
    audioCtx = new (window.AudioContext || window.webkitAudioContext)();
    if (audioCtx.state === 'suspended') await audioCtx.resume();
    const src = audioCtx.createMediaStreamSource(stream);
    analyser = audioCtx.createAnalyser();
    analyser.fftSize = 1024;
    src.connect(analyser);
    audioBuf = new Float32Array(analyser.fftSize);
  } catch (e) { /* mic denied: audio trigger simply never fires */ }
}

// RMS loudness of the current time-domain buffer, 0..~1.
function currentLoudness() {
  if (!analyser) return 0;
  analyser.getFloatTimeDomainData(audioBuf);
  let sum = 0;
  for (let i = 0; i < audioBuf.length; i++) sum += audioBuf[i] * audioBuf[i];
  return Math.sqrt(sum / audioBuf.length);
}

// Called once per frame; fires a spawn on an onset.
function pollAudio(nowMs) {
  const l = currentLoudness();
  loudBaseline = loudBaseline * 0.95 + l * 0.05;          // adaptive baseline
  const isOnset = l > CFG.loudFloor &&
                  l > loudBaseline * CFG.riseFactor &&
                  (nowMs - lastSpawnMs) > CFG.cooldownMs;
  if (isOnset) {
    lastSpawnMs = nowMs;
    const size = loudnessToSize(l);
    audioSpawn(size);
  }
}

function loudnessToSize(l) {
  const t = constrain((l - CFG.quiet) / (CFG.loud - CFG.quiet), 0, 1);
  return CFG.minSize + t * (CFG.maxSize - CFG.minSize);
}

// Position picker is added in Task 3; until then spawn near center.
function audioSpawn(size) {
  const w = size * (0.8 + Math.random() * 0.4);
  const h = size * (0.8 + Math.random() * 0.4);
  spawnShape(width / 2, height / 2, w, h);
}
```

- [ ] **Step 2: Start audio on run, poll each frame**

In `beginRun()`, after the `capture.hide();` line, add:

```javascript
  startAudio();
```

In `draw()`, inside the `appState === 'running'` path, replace the shapes loop region so it polls audio first. Change:

```javascript
  drawVideoCover();
  for (const s of shapes) { s.angle += s.speed; drawShapeCutout(s); }
  drawHud();
```

to:

```javascript
  pollAudio(millis());
  drawVideoCover();
  for (const s of shapes) { s.angle += s.speed; drawShapeCutout(s); }
  drawHud();
```

- [ ] **Step 3: Add a debug audio-onset simulator**

In `keyPressed()`, add a second key so onset logic is testable without a mic:

```javascript
  if (key === 'a' || key === 'A') audioSpawn(loudnessToSize(0.2));
```

So `keyPressed()` becomes:

```javascript
function keyPressed() {
  if (key === 's' || key === 'S') spawnShape(width / 2, height / 2, 160, 160, 'circle');
  if (key === 'a' || key === 'A') audioSpawn(loudnessToSize(0.2));
}
```

- [ ] **Step 4: Verify (Playwright MCP)**

Reload the page, start the run (Front, then center tap). Press `a` several times.
Expected: no console errors. If camera is available, the HUD "shapes:" count increases and centered rotating cutouts appear; if camera is denied, confirm no errors and that `audioSpawn` ran (the shapes array grows even though video isn't drawn — verify by adding nothing, just confirm no exceptions). Loudness math: `loudnessToSize(0.2)` with defaults = `60 + ((0.2-0.03)/(0.30-0.03))*300 ≈ 60 + 0.63*300 ≈ 249` px.

- [ ] **Step 5: Commit**

```bash
git add index-sensors.html
git commit -m "feat: audio onset trigger with loudness-driven size"
```

---

## Task 3: Position — tilt cursor blended with camera hotspot

**Files:**
- Modify: `index-sensors.html` (add orientation + hotspot modules, update `audioSpawn`)

- [ ] **Step 1: Add the orientation (tilt) module**

Insert before `/* ---------------- hud ---------------- */`:

```javascript
/* ---------------- orientation / tilt cursor ---------------- */
let neutralBeta = null, neutralGamma = null;   // captured at first reading
let rawBeta = 0, rawGamma = 0;
let tiltX = null, tiltY = null;                // smoothed cursor (screen px)

function onOrientation(e) {
  if (e.beta == null || e.gamma == null) return;
  if (neutralBeta == null) { neutralBeta = e.beta; neutralGamma = e.gamma; }
  rawBeta = e.beta; rawGamma = e.gamma;
}

function startOrientation() {
  const add = () => window.addEventListener('deviceorientation', onOrientation, true);
  if (typeof DeviceOrientationEvent !== 'undefined' &&
      typeof DeviceOrientationEvent.requestPermission === 'function') {
    DeviceOrientationEvent.requestPermission().then(r => { if (r === 'granted') add(); }).catch(() => {});
  } else { add(); }
}

// Smoothed tilt cursor in screen space; defaults to center until sensor data arrives.
function tiltCursor() {
  if (neutralBeta == null) return { x: width / 2, y: height / 2 };
  const dgx = constrain((rawGamma - neutralGamma) / CFG.tiltRange, -1, 1);
  const dgy = constrain((rawBeta  - neutralBeta)  / CFG.tiltRange, -1, 1);
  const targetX = width / 2 + dgx * (width / 2) * 0.9;
  const targetY = height / 2 + dgy * (height / 2) * 0.9;
  if (tiltX == null) { tiltX = targetX; tiltY = targetY; }
  tiltX += (targetX - tiltX) * CFG.tiltSmooth;
  tiltY += (targetY - tiltY) * CFG.tiltSmooth;
  return { x: tiltX, y: tiltY };
}
```

- [ ] **Step 2: Add the camera-hotspot module**

Insert directly after the orientation module:

```javascript
/* ---------------- camera hotspot ---------------- */
let hotBuf = null;                       // tiny offscreen for downsampled video
let hotspot = { x: 0, y: 0 };            // screen-space brightest/most-saturated cell

function scanHotspot() {
  if (!videoReady()) return;
  if (!hotBuf) { hotBuf = createGraphics(CFG.hotGridW, CFG.hotGridH); hotBuf.pixelDensity(1); }
  hotBuf.clear();
  hotBuf.image(capture, 0, 0, CFG.hotGridW, CFG.hotGridH);
  hotBuf.loadPixels();
  let best = -1, bx = CFG.hotGridW / 2, by = CFG.hotGridH / 2;
  const px = hotBuf.pixels;
  for (let gy = 0; gy < CFG.hotGridH; gy++) {
    for (let gx = 0; gx < CFG.hotGridW; gx++) {
      const i = 4 * (gy * CFG.hotGridW + gx);
      const r = px[i], g = px[i + 1], b = px[i + 2];
      const mx = Math.max(r, g, b), mn = Math.min(r, g, b);
      const sat = mx === 0 ? 0 : (mx - mn) / mx;
      const score = mx * (0.5 + sat);     // bright AND colorful
      if (score > best) { best = score; bx = gx; by = gy; }
    }
  }
  // Map grid cell -> screen px (mirror x for the front camera to match drawVideoCover).
  let nx = (bx + 0.5) / CFG.hotGridW;
  if (facing === 'user') nx = 1 - nx;
  hotspot.x = nx * width;
  hotspot.y = ((by + 0.5) / CFG.hotGridH) * height;
}
```

- [ ] **Step 3: Wire orientation start + throttled hotspot scan + blended spawn position**

In `beginRun()`, after `startAudio();` add:

```javascript
  startOrientation();
```

In `draw()`, in the running path, after `pollAudio(millis());` add the throttled scan:

```javascript
  if (frameCount % CFG.hotspotEveryN === 0) scanHotspot();
```

Replace `audioSpawn` to use the blended position:

```javascript
function audioSpawn(size) {
  const w = size * (0.8 + Math.random() * 0.4);
  const h = size * (0.8 + Math.random() * 0.4);
  const c = tiltCursor();
  const x = lerp(c.x, hotspot.x, CFG.blend);
  const y = lerp(c.y, hotspot.y, CFG.blend);
  spawnShape(x, y, w, h);
}
```

- [ ] **Step 4: Verify (Playwright MCP)**

Reload, start run. Dispatch a synthetic orientation event, then force a spawn:

Run (Playwright evaluate):
```javascript
window.dispatchEvent(new DeviceOrientationEvent('deviceorientation', { beta: 0, gamma: 0 }));
window.dispatchEvent(new DeviceOrientationEvent('deviceorientation', { beta: 0, gamma: 30 }));
```
Then press `a`.
Expected: no console errors; the spawn position shifts toward the right when gamma increases (tilt cursor moves), confirming the tilt path is live. (Hotspot stays centered without a real camera, which is the expected fallback.)

- [ ] **Step 5: Commit**

```bash
git add index-sensors.html
git commit -m "feat: tilt cursor + camera hotspot blended spawn position"
```

---

## Task 4: Shake-to-clear + spawn haptic confirm

**Files:**
- Modify: `index-sensors.html` (add motion module, wire start)

- [ ] **Step 1: Add the motion/shake module**

Insert before `/* ---------------- hud ---------------- */`:

```javascript
/* ---------------- motion / shake-to-clear ---------------- */
let lastShakeMs = -1e9;

function onMotion(e) {
  const a = e.accelerationIncludingGravity || e.acceleration;
  if (!a) return;
  const mag = Math.sqrt((a.x || 0) ** 2 + (a.y || 0) ** 2 + (a.z || 0) ** 2);
  const nowMs = millis();
  if (mag > CFG.shakeThreshold && (nowMs - lastShakeMs) > CFG.shakeCooldownMs) {
    lastShakeMs = nowMs;
    shapes = [];
    if (navigator.vibrate) navigator.vibrate(CFG.vibrateClear);
  }
}

function startMotion() {
  const add = () => window.addEventListener('devicemotion', onMotion, true);
  if (typeof DeviceMotionEvent !== 'undefined' &&
      typeof DeviceMotionEvent.requestPermission === 'function') {
    DeviceMotionEvent.requestPermission().then(r => { if (r === 'granted') add(); }).catch(() => {});
  } else { add(); }
}
```

- [ ] **Step 2: Wire motion start**

In `beginRun()`, after `startOrientation();` add:

```javascript
  startMotion();
```

- [ ] **Step 3: Verify shake clears (Playwright MCP)**

Reload, start run, press `a` a few times to add shapes. Then dispatch a hard motion event:

Run (Playwright evaluate):
```javascript
window.dispatchEvent(new DeviceMotionEvent('devicemotion', {
  accelerationIncludingGravity: { x: 30, y: 30, z: 30 }
}));
```
Expected: no console errors; the HUD "shapes:" count resets to 0 (magnitude √(2700) ≈ 52 > threshold 28). Note: synthetic `DeviceMotionEvent` construction is supported in Chromium; if the constructor rejects the init dict, fall back to `Object.assign(new Event('devicemotion'), {...})` in the test only.

- [ ] **Step 4: Commit**

```bash
git add index-sensors.html
git commit -m "feat: shake-to-clear with haptic confirm"
```

---

## Task 5: On-device run notes + final pass

**Files:**
- Modify: `index-sensors.html` (HUD hint + header comment for ngrok)
- Modify: `docs/superpowers/plans/2026-06-03-phone-sensor-sketch.md` (mark done)

- [ ] **Step 1: Add an unobtrusive first-run hint to the HUD**

Replace `drawHud()` with:

```javascript
function drawHud() {
  fill(255, 150); textSize(13); textAlign(LEFT, BOTTOM);
  text('shapes: ' + shapes.length, 14, height - 12);
  if (shapes.length === 0) {
    textAlign(CENTER, CENTER); fill(255, 120);
    text('make a sound to draw • tilt to aim • shake to clear', width / 2, height - 24);
  }
}
```

- [ ] **Step 2: Document the ngrok serving path in the header comment**

In the top `/* ... */` comment block, append before the closing `*/`:

```
  Run on phone: from the project dir start a static server
  (python3 -m http.server 8000), expose it with `ngrok http 8000`, then open the
  https URL ngrok prints on the phone. Camera + mic + motion require this HTTPS
  context; a plain file:// will not grant them.
```

- [ ] **Step 3: Full desktop smoke test (Playwright MCP)**

Reload, start run, exercise all debug paths: `s`, `a` (×several), synthetic orientation (gamma sweep), synthetic hard motion.
Expected: shapes spawn and rotate, position responds to tilt, shake clears, no console errors across the whole sequence.

- [ ] **Step 4: On-device confirmation (manual, user-run)**

Start the server + ngrok, open the https URL on the Android phone, tap to start, grant camera/mic/motion. Confirm: a clap spawns a sized cutout, tilting aims it, the live video spins inside, a hard shake clears. Tune `CFG` values on-device as needed (audio thresholds and `tiltRange` are the likely candidates).

- [ ] **Step 5: Commit**

```bash
git add index-sensors.html docs/superpowers/plans/2026-06-03-phone-sensor-sketch.md
git commit -m "feat: on-device hint + ngrok run notes; finalize sensor sketch"
```

---

## Self-Review Notes

- **Spec coverage:** start screen + camera pick (Task 1), audio onset trigger + loudness→size (Task 2), tilt+hotspot position (Task 3), constant spin / no added color (Task 1 render core, unchanged), infinite buildup (no removal logic anywhere), shake-to-clear + haptics (Tasks 1 & 4), ngrok/secure-context + perf via pixelDensity(1) and throttled scan (Tasks 1, 3, 5). All covered.
- **No real unit tests:** the repo has no test runner and sensors can't be meaningfully unit-tested; verification is Playwright + synthetic events + debug keys (`s`, `a`) plus final on-device manual confirmation. This is the honest verification path for a browser sketch.
- **Type consistency:** `spawnShape(x, y, w, h, type?)`, `audioSpawn(size)`, `tiltCursor()→{x,y}`, `hotspot{x,y}`, `CFG.*` names are consistent across tasks.
```
