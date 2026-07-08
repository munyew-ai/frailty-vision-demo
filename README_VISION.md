# Vision Demo — Chair-Stand Rep Counter

## What it does and why it's standalone

`vision-demo.html` is a single-file, zero-dependency chair-stand (sit-to-stand) repetition counter built with the MediaPipe Tasks Vision SDK. It runs entirely in the browser with no build step, no server, and no network calls after the initial model download. Being standalone lets clinicians or developers open it directly from a file system or host it as a static asset — no Node.js, no bundler, no backend needed. It was designed as a rapid-integration prototype for the Frailty Coach project.

## MediaPipe Tasks Vision approach (WASM, on-device)

The file loads `@mediapipe/tasks-vision` (v0.10.14) via an importmap pointing to jsDelivr CDN. On first load, the browser fetches the WASM binaries and the `pose_landmarker_lite` model (~6 MB) from Google's storage bucket. After that initial fetch, everything runs locally:

- **GPU delegate** is requested first; falls back to CPU automatically.
- `PoseLandmarker.detectForVideo()` is called on every animation frame, yielding 33 normalized (x, y, z, visibility) landmark coordinates.
- Only the two hip landmarks (indices 23 and 24 — `LEFT_HIP` and `RIGHT_HIP`) are used for rep counting. The full skeleton is drawn on a canvas overlay for visual feedback.

All video data stays on-device. The privacy banner ("🔒 All processing on-device — no video leaves your phone") reflects this architectural guarantee.

## Rep detection state machine

The state machine tracks a single `hipY` value — the average normalized Y coordinate of the left and right hips (0 = top of frame, 1 = bottom).

**Thresholds:**
| Threshold | hipY value | Meaning |
|-----------|-----------|---------|
| Sitting   | > 0.55    | Hips low in frame |
| Standing  | < 0.42    | Hips high in frame |
| In-between | 0.42–0.55 | Transitioning |

**State transitions:**
```
idle → sitting → rising → standing → lowering → sitting (rep++)
```

A rep is counted when the machine re-enters `sitting` after having been in `standing` or `lowering`. Partial attempts (e.g. starting to rise but sitting back before fully standing) do not increment the counter.

**Debounce:** any threshold must be held for ≥ 5 consecutive frames before the state machine commits to a transition. At 30 fps this is ~167 ms — enough to filter noise without adding perceptible lag.

## How to integrate into the React app

**Option A — iframe embed (fastest):**
```jsx
<iframe
  src="/vision-demo.html"
  style={{ width: "100%", height: "680px", border: "none", borderRadius: "16px" }}
  allow="camera"
  title="Chair Stand Counter"
/>
```
Pass the `repCount` out via `postMessage`:
```js
// Inside vision-demo.html, after repCount++:
window.parent.postMessage({ type: "rep", count: repCount }, "*");
```
```jsx
// In React parent:
useEffect(() => {
  const handler = (e) => { if (e.data.type === "rep") setReps(e.data.count); };
  window.addEventListener("message", handler);
  return () => window.removeEventListener("message", handler);
}, []);
```

**Option B — React hook (recommended long-term):**
Extract the landmark processing logic (`processLandmarks`, `applyDebounce`, `transitionTo`) into a `useChairStandCounter(videoRef)` hook that accepts a video element ref and returns `{ repCount, state, feedback }`. Wire the MediaPipe detection loop inside a `useEffect`.

## Known limitations / next steps

- **Sample video is not a real chair-stand clip.** Replace `src` with an actual chair-stand exercise video for meaningful sample-mode testing.
- **Single-person only.** The model is configured for `numPoses: 1`; crowded environments will use the first detected pose.
- **Hip-Y threshold is camera-angle sensitive.** Thresholds were tuned for a side-on or slight 45° view. A frontal view may require different values or a more robust angle-invariant calculation (e.g., using knee Y relative to hip Y).
- **No calibration step.** A short "sit then stand" calibration pass could auto-tune thresholds per user.
- **Accessibility.** An audio cue (Web Audio API beep) on each completed rep would benefit users with visual impairments.
- **PWA / offline use.** Adding a service worker to cache the WASM and model blobs would make the tool work fully offline after one network fetch.
