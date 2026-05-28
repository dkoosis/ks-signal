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

1. **Primary — richer retrieval (FOCUS).** Attach situational state to each memory so retrieval
   can match on *"what state was I in when I wrote this,"* not semantic similarity alone.
2. **Secondary — coaching.** Surface patterns over time (fatigue, stress, focus drift).

### Why the psychomotor layer earns its place (retrieval justification)

Operational context (app, project, time) is the obvious retrieval index — and trixi mostly has
it already. The keystroke layer's *unique* contribution is **affect-matched retrieval**:
"surface what I was doing *last time I was in this state*" (scattered / fatigued / wired),
**regardless of topic**. Semantics can't do it; operational tags only crudely approximate it.
That how-was-I dimension is the whole reason ks-signal exists; everything is justified against it.

### Calibration method — opportunistic labels (firm)

The lab validates cheap always-on keystroke features against **rare, high-quality ground-truth
labels** captured opportunistically:
- **Transcripts** (Claude logs) → task + sentiment labels.
- **HRV** (Polar, worn *sometimes*) → arousal labels. Not a runtime signal — too intermittent —
  but during overlap windows it is ground truth for "which keystroke features track arousal," so
  keystroke-only can proxy arousal when the Polar is off.

Method: extract candidate features → align with labels during overlap windows → keep features
that predict, drop the rest → survivors become the baseline + featurespec. This is what
"lab-first" concretely means.

### Signal landscape (context; mostly NOT ks-signal's job)

The full situational fingerprint is multi-signal and **composed by trixi** (integration is
trixi's job, confirmed). trixi already captures location/weather/calendar/email/mood-via-language;
much else is derivable from existing logs. New sensors are added one at a time only when the lab
proves a gap. Candidate next sensor (parked): **app/window focus switching** ("focus2ks") —
cheap, privacy-clean, strong focus-vs-scatter proxy. Voice: parked (see below). Mood: trixi
infers from language; no sensor.

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

## Findings

- **F1 (2026-05-28): dictation is invisible to the keystroke stream.** Measured today's
  it2ks log while dk dictated extensively via Wispr: <0.4% of key-down gaps are
  sub-physiological; 83% are 50–500ms human-typing range. Wispr inserts via
  paste/accessibility, not synthetic keystrokes, so iTerm2 emits no notifications for it.
  → dictation cannot pollute the typing baseline (good), and cannot be captured *or* inferred
  from keystrokes alone (it looks identical to idle). Capturing modality is a **capture-layer**
  gap, not a ks-signal one.

- **F2 (2026-05-28): dictation IS recoverable by transcript correlation — spike succeeded.**
  The dictated text it2ks misses still reaches Claude and sits in
  `~/.claude/projects/.../*.jsonl` with timestamps. Correlating user-message timestamps with
  it2ks key-down presence cleanly separates modality: dictated messages show **0.01–0.05
  keydowns/char**, typed show **0.5–1.3** — an order-of-magnitude gap, single-threshold
  separable. No mic, no accessibility, no new capture. Caveats: coverage is **Claude-context
  only** (dictation into other apps still invisible); couples to Claude's undocumented log
  format (**lab-only**, never the runtime contract).

## Direction: voice channel — PARKED

Detecting dictation was briefly framed as a router for a **voice** sensor (Layer 1b). The F2
correlation makes voice unnecessary for the near term: the transcript yields both the modality
tag *and* the richer operational/situational context (the conversation = what dk was doing),
which is closer to the primary goal than voice prosody and already exists on disk. Voice is
parked, possibly permanently. If ever revived: extract prosodic features locally, NEVER store
or transmit audio.

## Transcript correlation — placement (firm)

- **Runtime = trixi's job.** Composing keystroke-state with "what was I doing" is signal
  *integration* → trixi owns it. The Claude transcript is a context source trixi reads, like
  weather/location. ks-signal does NOT read Claude logs at runtime (would collapse the tier
  boundary + couple to an unstable format).
- **Lab = ks-signal, offline.** ks-signal MAY use transcripts as *labels* to discover which
  keystroke features track states and to calibrate baselines. Findings work only — never the
  snapshot, never the featurespec.

## Open

- **Process model** — continuous daemon vs on-demand recompute vs **lab-first**. Lean:
  lab-first, because the runtime normalizer depends on features the lab must first discover
  (using transcript labels per F2). Resolve next, before file/daemon layout.

## Input contract (from it2ks)

Newline-delimited JSON under `~/.it2ks/logs/YYYY-MM-DD.jsonl`. Session records
(`s`, `sid`, `app`, `t0`) + keystroke records (`ts` ns-monotonic, `wall`, `s`, `act`
down/up/flags, `key` keycode, `char`). Dwell recoverable (down→up); expect fewer ups than
downs (auto-repeat, modifier-only `flags`).
