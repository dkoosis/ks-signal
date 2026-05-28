# ks-signal North Star

status: drafting (brainstorm 2026-05-28)
related: it2ks (`~/Projects/it2ks`), trixi placement proposal `~/Projects/trixi/docs/proposals/2026-05-28-keystroke-analytics-placement.md`

## What ks-signal is

The **analytics tier** between capture (it2ks) and runtime (trixi/mnemosyne). It turns raw
keystroke events into a **psychomotor state descriptor** — mnemosyne's situational
fingerprint, **Layer 1**.

```
it2ks (capture) → ks-signal (analytics) → trixi (runtime)
   JSONL logs        state descriptor        retrieval cue
```

## Goals (ranked)

1. **Primary — richer retrieval.** Attach situational state to each memory so retrieval can
   match on *"what state was I in when I wrote this,"* not semantic similarity alone.
   Keystrokes are one input to that broader emotional/operational/situational context.
2. **Secondary — coaching.** Surface patterns over time (fatigue, stress, focus drift).

## Scope boundary (firm)

- ks-signal owns the **keystroke → state** transform only. It is Layer 1 (psychomotor).
- It does **not** know about nugs, calendars, tasks, or what dk is working on. Broader
  situational context = other signals / other layers.
- **Baseline first.** No universal keystroke stress markers exist. The signal is *deltas
  from dk's personal baseline*, conditioned per (app, time-of-day). Calibration precedes
  any usable fingerprint.
- **Build above capture, not inside it.** it2ks stays capture-only. Same JSONL schema lets
  linux2ks / win2ks feed ks-signal later without renegotiation.

## Interface (firm direction)

- **State file, pulled by the consumer** — mirrors the existing it2ks → trixi model.
  trixi reads an up-to-date state file at nug-write, attaches it, moves on. Missing file =
  no payload, no error ("IFF available" is structural, not defensive).
- trixi imports **no** ks-signal code. The state file is self-describing, versioned JSON;
  the consumer stores keys it does not interpret. This **supersedes** the placement
  proposal's "vendored `featurespec` Go package, no runtime data flow" coupling model.

## Working assumptions (NOT yet ratified — open in brainstorm)

- **Baseline-deviation is the product; raw features are intermediate.** A retrieval key must
  be comparable across contexts, so the fingerprint is expressed as deviation-from-baseline
  (e.g. z-scores), not raw feature values.
- **Snapshot-in-time is sufficient.** ks-signal persists only baselines + current state.
  Per-nug snapshots accumulate into the distributed history that serves retrieval; coaching
  recomputes longitudinal views from raw logs on demand (logs are the durable time series).
  → No separate time-series store, yet.
- Process model (continuous daemon vs on-demand recompute vs lab-first) — **undecided.**

## Input contract (from it2ks)

Newline-delimited JSON under `~/.it2ks/logs/YYYY-MM-DD.jsonl`. Session records
(`s`, `sid`, `app`, `t0`) + keystroke records (`ts` ns-monotonic, `wall`, `s`, `act`
down/up/flags, `key` keycode, `char`). Dwell recoverable (down→up); expect fewer ups than
downs (auto-repeat, modifier-only `flags`).
