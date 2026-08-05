# Clap Trainer

A browser-based rhythm trainer. The app plays a quarter-note metronome and you clap the
subdivisions — eighth notes or triplets. Your microphone picks up each clap and the game
scores it against the beat grid with millisecond feedback: early, late, good (±50 ms),
great (±20 ms), or missed.

**▶ Play it here: [dsiegel.github.io/clapping_game](https://dsiegel.github.io/clapping_game/)**

## Features

- Metronome and scoring share one Web Audio clock, so audio, visuals, and scoring agree.
- Mic clap detection with automatic latency compensation — no calibration step.
- DDR-style falling note lane: notes cross the target line exactly on the beat, and your
  claps drop colored ticks into the same stream so relative timing is directly visible.
- Level progression that gradually removes visual support, ending with random
  eighth/triplet switches you must adapt to by ear.
- Offset and detection charts to expose systematic drift and tune sensitivity.

## Running locally

Everything is a single dependency-free `index.html`. The mic requires a secure context,
so serve it over localhost:

```sh
python3 -m http.server 8123
```

then open <http://localhost:8123>.
