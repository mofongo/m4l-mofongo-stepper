# Mofongo Stepper — Metric-Sync Polymetric Step Sequencer

**Date:** 2026-07-12
**Type:** Max for Live MIDI Effect (`.amxd`)
**Status:** Approved design, ready to build

## Purpose

A trigger-driven melodic step sequencer for Ableton. A percussive trigger (e.g. a
snare on another track, routed in via native Ableton MIDI routing) advances a master
step counter through a probability gate. Each advance reads a stored pitch from a
`coll` and plays it down Track 2's instrument chain. Loop length is modulated by a
5-stage length modifier, and the counter periodically re-syncs to the Ableton
timeline at bar boundaries.

This is a NEW device, built into a clean blank patcher. The existing `sonogrid`
device in the other patcher is left untouched.

## Approved deviations from the source spec

- **D1** — Metric clock uses message-domain `transport` (driven by `metro`) instead
  of `plugsync~` + `qmetro 20`. Bar-boundary resets do not need sample accuracy;
  this avoids signal→control conversion and matches the existing project idiom.
- **D2** — `counter` reset and max-count inlets/messages are taken from
  `get_object_doc counter` rather than the spec's assumed "4th inlet / 5th inlet"
  numbering. Behavior identical to the spec's intent.
- **D3** — UI is built from native `live.*` objects (no custom `jsui`). The 16-step
  pitch matrix is a **row of 16 `live.numbox`** (one MIDI note per step).

## Architecture

```
midiin → midiparse ─┬─(pitch/vel)→ [== TargetNote] → [vel ≥ Threshold] → [random 100 < Prob] → sel 1
                    │                                                                    │ ADVANCE bang
                    │                                        counter 1 16 (master) ──────┘
                    │                                             ↑reset   ↑maxcount
                    │                                     metric-reset   5-stage length mod
                    │                                                          │ current step index
                    │                                                          ↓
                    │                                              coll (step→pitch) → makenote → midiformat → midiout
                    └─(local pitches)→ stripnote → gate(StepRec) → pack(pitch,WritePointer) → coll (write)
```

### Module A — Signal filtering & conditioning
- `midiin → midiparse`; note pitch/velocity via the note outlet → `unpack`.
- Target-Note selector: `== <TargetNote>` (from `live.numbox`, MIDI-note format).
- Velocity threshold gate: `>= <Threshold>` (0–127). Note-offs (vel 0) blocked by any
  threshold > 0, preventing double-trigger.

### Module B — Random probability engine
- On a qualifying trigger: `random 100` → `< <Prob>` (0–100 from `live.dial`) → `sel 1`.
- Only a success emits the ADVANCE bang.

### Module C — Main sequencer engine & metric reset
- Master driver: `counter 1 16`, advanced by the ADVANCE bang.
- Pitch store: `coll` mapped step→pitch (0–15).
- Metric clock (D1): `metro` → `transport`; beats → `trunc` → `change` →
  `% <Interval>` → `sel 0` → counter reset. Interval from `live.tab`:
  4 = 1 bar, 6 = 1.5 bars, 8 = 2 bars.

### Module D — 5-stage length modifier & quantized bypass latch
- Secondary `counter 1 5`, advanced by the master counter's carry-out.
- 5× `live.numbox` (1–16) = per-stage loop lengths; selected by a `switch`/`coll`.
- Bypass: a `switch 2` chooses static Default-Length knob vs. the 5-stage stream.
- Anti-glitch: toggle choice buffered in an `int`, updated ONLY on the master
  counter's carry-out bang, so `switch` state flips exactly on the loop boundary.
- `switch` output → counter max-count inlet/message (D2).

### Module E — Step recording engine
- `live.text` "Step Rec" toggles a routing `gate`.
- Local input: `midiparse → stripnote → gate`.
- On a recorded note: `pack <pitch> <WritePointer>` → write to `coll`.
- Auto-advance: `t b b` / `pipe` bumps WritePointer after each write.
- Entering Step-Rec resets WritePointer to step 1.

### Module 3 — UI (native live.*), three zones
1. **Input conditioning:** Target Note, Velocity Threshold, Probability.
2. **Sequencer matrix:** row of 16 `live.numbox` (D3) + 5 stage `live.numbox`.
3. **Global & modes:** metric-reset `live.tab`, bypass toggle, Step-Rec toggle,
   Master Panic Reset button.
- Status: `live.button` flasher on attempted triggers; `live.led`s tracking the
  active playing step and active modifier stage.

## Build order (each stage verified live in Max before the next)

1. Core playable loop: input → gates → `counter` → `coll` → `makenote` → out.
2. Metric reset (Module C clock + modulo + reset).
3. 5-stage length modifier + quantized bypass latch (Module D).
4. Step-Rec engine (Module E).
5. UI polish + LED/flasher status indicators.

## Routing / muting notes
- Single M4L MIDI input: both the routed trigger and local track notes arrive at one
  `midiin`; they are separated by pitch (Target-Note selector vs. everything else).
- Trigger notes are consumed by the routing logic and NOT passed to Track 2's output.
```
