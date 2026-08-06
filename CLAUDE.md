# Clapping Game

A browser-based rhythm trainer. The app plays a quarter-note metronome (4/4, default 57 BPM)
and the player claps the subdivisions — eighth notes ("1 & 2 &") or triplets
("1-trip-let 2-trip-let"). The microphone picks up each clap and the game scores it against
the expected grid, telling the player whether they were early, late, good (within ±50 ms),
great (within ±20 ms), or missed the note entirely.

## Goals

- Give immediate, precise timing feedback (millisecond offsets, not just pass/fail).
- Train subdivision feel progressively: start with full visual support, then remove it as
  the player proves consistency, ending with random mode switches they must adapt to by ear.
- Stay dependency-free: one HTML file, Web Audio API only, no build step, no server logic.

## Key features

- **Metronome + scoring on one clock.** Clicks are scheduled on the Web Audio clock and the
  scoring grid is derived from the same timestamps, so audio, visuals, and scoring agree.
- **Mic clap detection.** The mic runs through two 3 kHz high-pass filters (cutting the
  660/990 Hz metronome bleed ~40 dB while a clap's broadband snap passes) into an
  AudioWorklet that detects transients: a block's peak must clear the sensitivity floor
  AND jump 4× above a peak-hold envelope of the recent level (~40 ms decay). Clap peaks
  feed the envelope so their tails can't re-trigger; a 120 ms refractory period and
  sample-accurate timestamps round it out. No calibration step needed.
- **Spacebar input.** SPACE always registers as a clap while playing (compensated by output
  latency only — a keypress never crosses the mic path). A "Use microphone" checkbox in the
  panel disables the mic entirely: unchecked at Start, `getUserMedia` is skipped (no
  permission prompt, no calibration blips or count-in warning); unchecked mid-game, mic
  detections are ignored live while spacebar keeps working.
- **Latency compensation.** Count-in clicks carry an extra 6 kHz blip that passes the
  detector's high-pass, so the app hears its own clicks through the mic and measures true
  round-trip latency (median of detected − scheduled). Scoring subtracts that from each
  detection; if no blip comes back (headphones), it falls back to the browser's reported
  output + mic input latency. The detection chart legend shows the value and its source.
- **DDR-style falling lane.** Fullscreen canvas; notes fall from the top and cross the target
  line at the screen's vertical midpoint exactly on the beat. Claps drop colored ticks into
  the same stream, misses drop red ✕ marks, so relative timing is directly visible.
- **Level progression.** A rolling window of the last 12 results drives promotion/demotion:
  1. all notes shown → 2. subdivisions hidden → 3. beats hidden → 4. random eighths/triplets
  in 2–4-measure segments (mode can only change on a "1"). Checkboxes enable/disable
  individual levels;
  progression skips disabled ones and effects stack — each level's effect applies only if
  that level is enabled and reached (level 3 unchecked → beats stay visible at level 4).
  The current level is highlighted. Triplet measures tint the whole canvas violet.
- **Offset chart.** A strip along the bottom plots the last 60 s of clap offsets (early above,
  late below, ±50 ms band) with an EWMA overlay (α = 0.25) to expose systematic drift.
- **Detection chart.** A second strip above the offset chart plots the detector's internals
  for the last 12 s (sqrt-scaled): mic peak post-filter (grey), the env ×4 transient
  threshold (orange), the sensitivity floor (dashed blue), and a green tick per detection —
  so a missed clap shows exactly which threshold its spike failed to clear.

## Layout

Everything lives in `index.html`: a fullscreen `<canvas>`, a floating control panel on the
left (BPM, mode, levels, mic toggle, sensitivity, Start), a HUD overlay at top center
(current mode banner, level, verdict, offset, stats), big current-streak counters at top
right, and a best-streaks line at bottom right just above the detection chart. Serve over localhost — `getUserMedia`
needs a secure context (`python3 -m http.server 8123`).

## Function index (all in `index.html`)

### Audio plumbing
- `WORKLET_CODE` — AudioWorkletProcessor source: per-block peak scan, transient detection
  against the envelope + floor. Posts `{type:'clap', t}` on detection and `{type:'lvl',
  t, peak, env}` level telemetry every ~20 ms (for the detection chart).
- `loadWorklet()` — registers `WORKLET_CODE` via a data: URL (Chrome rejects blob: worklet
  modules on file://, so a data: URL is the one loader that works everywhere).
- `ensureAudio()` — creates the AudioContext, opens the mic (echo cancellation/AGC off),
  and wires mic → high-pass ×2 → clap-detector worklet.
- `sensFloor()` — detection floor from the slider, inverted so right = more sensitive.
- `sendFloor()` — pushes the current floor to the worklet (live, on input).
- `teardownAudio()` — releases the mic and closes the context (only on setup failure;
  `stop()` merely suspends the context so restarting never re-prompts for the mic).
- `latComp()` — browser-reported output + mic input latency (the fallback estimate).
- `effLat()` — effective compensation: the count-in loopback measurement when available,
  else `latComp()`.
- `click(time, accent, bright)` — schedules one metronome blip (990 Hz accented, 660 Hz
  otherwise; 4 ms attack ramp — pitched/shaped so speaker distortion harmonics stay under
  the 3 kHz high-pass). `bright` (count-in only) adds the 6 kHz latency-measurement blip.

### Game state & scoring
- `start()` — resets state, starts the click scheduler (also queues `expected` grid points
  per measure), kicks off the draw loop.
- `stop()` — tears everything down and returns to idle.
- `onClap(t)` — snaps a detected clap to the nearest grid point for its measure's mode,
  computes the ms offset, claims the expected grid point, records history/EWMA, updates HUD.
- `onMiss(ex)` — fires when an expected grid point passes unclaimed: HUD "MISS", red ✕,
  counts as a bad result.
- `updateStats()` — refreshes the good/missed/average line in the HUD.
- `updateStreaks()` — refreshes the big current-streak counters at top right (consecutive
  good-or-better claps, and great-only claps; a miss or off-grid clap resets both, a merely
  good clap resets the great streak) and the best-streaks line at bottom right.

### Modes & levels
- `getSubDiv()` — current radio selection (2 = eighths, 3 = triplets), read live.
- `modeForMeasure(n)` — the subdivision governing measure *n*: the radio choice below
  level 4, a cached random pick at level 4 held for 2–4-measure segments (so
  drawing/scoring/banner agree and segments aren't over in a single measure).
- `enabledLevels()` — levels whose checkboxes are on (falls back to [1]).
- `effectActive(l)` — whether level *l*'s effect applies: enabled AND current level ≥ l.
- `setLevel(l, flash)` — switches level, resets the scoring window, locks the in-progress
  measure's mode when entering level 4.
- `recordResult(wasGood)` — rolling 12-result window; ≥10 good promotes to the next enabled
  level, ≤6 good demotes. The first 8 results after a level change are ignored (grace
  period) so adjustment misses can't demote right back — in triplets one measure alone
  is 12 results.
- `updateLevelHUD(flash)` — level text in the HUD + highlight on the current level checkbox.

### Rendering
- `resize()` — sizes the canvas to the window with devicePixelRatio scaling.
- `drawViz()` — per-frame loop: advances miss detection, draws clap ticks, miss ✕s,
  subdivision notes, main beats, target line, mode banner; calls `drawChart`. During the
  count-in it pulses a big "QUIET! CALIBRATING!" warning (clapping there would pollute
  the latency measurement).
- `drawChart(now)` — bottom strip: 60 s of offset dots, miss marks, ±50 ms band, EWMA line.
- `drawDetChart(now)` — detection strip above it: 12 s of worklet telemetry (mic peak,
  env ×4 threshold, sensitivity floor, green detection ticks), sqrt-scaled.
- `drawTargetLine(flash)` / `drawIdle()` — the midline (flashes on each beat) and idle screen.
