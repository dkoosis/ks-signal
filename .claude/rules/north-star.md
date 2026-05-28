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

## Smart sensor, not the self-model (firm)

ks-signal is a **smart sensor** — like a body-composition scale with per-user profiles, not
a dumb scale emitting raw kg. It knows your *keyboard*; it emits both a raw reading and a
normalized one. **trixi knows *you*** — it owns the model of dk by composing keystroke-state
with weather, location, mood, calendar, etc. The division:

- **ks-signal owns the keystroke *normalizer*.** The baseline ("normal for dk") must be built
  from the *full* keystroke stream, and only ks-signal sees that stream (it tails every log).
  A baseline built from nug-write moments alone would be a biased sample — "dk *while writing
  nugs*," not dk's typing baseline. So baseline lives here by **necessity**, not preference.
- **trixi owns the model of dk** at the integration layer, and records psychometric metrics
  (raw features) as a time series the same way it records weight / location / weather.

## Interface (firm direction)

- **State file, pulled by the consumer** — mirrors the existing it2ks → trixi model.
  trixi reads an up-to-date state file at nug-write, attaches it, moves on. Missing file =
  no payload, no error ("IFF available" is structural, not defensive).
- trixi imports **no** ks-signal code. The state file is self-describing, versioned JSON;
  the consumer stores keys it does not interpret. This **supersedes** the placement
  proposal's "vendored `featurespec` Go package, no runtime data flow" coupling model.
- **Snapshot payload carries both** (not either/or):
  - **raw features** (IKI, dwell, backspace rate, pause counts) → trixi logs these as
    psychometric metrics for coaching / longitudinal / cross-signal correlation;
  - **deviation** (z-scores vs baseline) → the immediately-comparable retrieval key, since
    trixi cannot correctly normalize keystrokes itself (it lacks the full stream);
  - **subject block** — who this is + confidence (see below).

## Subjects & baseline hygiene (firm)

- **Baseline is per-subject.** Every window is attributed before it folds into a baseline.
- **Single-subject verification only** (v1): "is this window consistent with the enrolled
  primary (dk)? yes / no / unsure." Deliberately a **low bar** — identity is not the purpose.
- **Filter non-human input first.** On a dev machine the dominant "not-dk" source is injected
  input — paste bursts, **STT/dictation (e.g. Wispr)**, auto-repeat, scripted input — not
  guest humans. It is frequent (dk dictates) and cheap to reject (sub-physiological timing,
  no plausible dwell distribution, machine regularity). This gate runs before primary-vs-not.
- **Cold-start by phasing.** During calibration dk is the presumed primary by being the
  dominant typist; bootstrap the primary profile from the calibration window, then switch
  verification on once the profile exists.
- **Deferred (YAGNI):** multi-profile enrollment, telling guest A from guest B,
  cluster-discovery of new subjects. Build the verifier now; add profiles when a second real
  subject appears.
- **Parked idea:** tag `modality: dictated` as positive situational context rather than
  silently dropping injected windows.

## Snapshot-in-time is sufficient (firm)

ks-signal persists only **baselines + current state**. Per-nug snapshots accumulate into the
distributed history that serves retrieval; coaching recomputes longitudinal views from raw
logs on demand (logs are the durable time series). → **No separate time-series store, yet.**

## Open

- **Process model** — continuous daemon vs on-demand recompute vs lab-first. Undecided;
  resolve after fundamentals, before file/daemon layout.

## Input contract (from it2ks)

Newline-delimited JSON under `~/.it2ks/logs/YYYY-MM-DD.jsonl`. Session records
(`s`, `sid`, `app`, `t0`) + keystroke records (`ts` ns-monotonic, `wall`, `s`, `act`
down/up/flags, `key` keycode, `char`). Dwell recoverable (down→up); expect fewer ups than
downs (auto-repeat, modifier-only `flags`).
