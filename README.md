# Repro: `--fps 60` renders GSAP/DOM motion at 30fps

Minimal reproduction for: the render-seek path re-quantizes every seek to a hardcoded
`canonicalFps = 30`, so GSAP/DOM motion only updates 30 times per second even when rendering
at `--fps 60`. The output is a correct 60fps CFR file, but every other frame is a duplicate.

[`index.html`](index.html) is the blank `hyperframes init` scaffold with a single change: one box
that moves **linearly** across the screen over 4s. At a true 60fps every frame must differ.

## Reproduce

```bash
npx hyperframes@0.7.10 render . -o repro_60fps.mp4 --fps 60 --workers 1
npx hyperframes@0.7.10 render . -o repro_30fps.mp4 --fps 30 --workers 1   # control
```

`--workers 1` keeps the capture single-threaded so the cadence is clean to measure; the bug
reproduces with the default worker count too.

## Observe

Play `repro_60fps.mp4` — the box visibly stutters (judders) instead of gliding. `repro_30fps.mp4`
is perfectly smooth for its rate.

Quantify with per-frame luma delta (every frame should differ at a true 60fps):

```bash
ffmpeg -i repro_60fps.mp4 \
  -vf "select='between(n,60,99)',signalstats,metadata=print:key=lavfi.signalstats.YDIF" \
  -an -f null - 2>&1 | grep -oE 'YDIF=[0-9.]+'
```

**Actual (60fps file):** the value alternates `motion, 0, motion, 0, …`

```
0.317 0.000 0.317 0.000 0.317 0.000 0.317 0.000 ...
```

Every other frame has zero motion delta — a duplicate. That is 30fps motion in a 60fps file.

Exact duplicate-frame count:

```bash
ffmpeg -i repro_60fps.mp4 -f framehash - 2>/dev/null | grep -vE '^#' \
  | awk '{if($NF==p)d++;p=$NF;t++}END{printf "%d frames, %d exact dups\n",t,d}'
```

- `repro_60fps.mp4` → 240 frames, ~104 exact consecutive duplicates (≈30fps of real motion).
- `repro_30fps.mp4` → 120 frames, **0** duplicates (30fps renders correctly).

> Note: naive duplicate detectors (`mpdecimate`) and full-frame hashing can under-report on real
> compositions, because a static grain overlay or font anti-aliasing makes otherwise-frozen frames
> differ by a hair. The per-frame `YDIF` delta above is the reliable signal — on this bare repro it
> is unambiguous.

## Root cause (pointers into the repo)

- `packages/core/src/runtime/player.ts` — `renderSeek` → `seekTimelineDeterministically` calls
  `quantizeTimeToFrame(timeSeconds, canonicalFps)`.
- `packages/core/src/inline-scripts/parityContract.ts` — `quantizeTimeToFrame` floors onto the fps
  grid: `Math.floor(t * fps) / fps`.
- `packages/core/src/runtime/state.ts` — `canonicalFps: 30` is the only production assignment; there
  is no setter, no `data-fps` attribute, and the render `--fps` is never wired into it. So output
  frame `k` (time `k/60`) seeks to `floor(k/2)/30` → frames `(0,1)`, `(2,3)`, … collapse in pairs.
- The WebGL shader-transition path is exempt because it samples `__HF_VIRTUAL_TIME__` at the exact
  `t` (`packages/producer/src/services/fileServer.ts`, the `hf.seek` wrapper).

## Expected

`--fps 60` should sample the timeline on a 60fps grid (drive `canonicalFps` from the render fps),
so every output frame is a distinct timeline position — matching the Studio preview, which plays in
real time and is smooth.
