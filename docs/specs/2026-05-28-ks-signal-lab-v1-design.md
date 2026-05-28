# ks-signal — Lab v1 design

date: 2026-05-28
status: draft for review
related: `.claude/rules/north-star.md`, it2ks (`~/Projects/it2ks`),
trixi placement proposal `~/Projects/trixi/docs/proposals/2026-05-28-keystroke-analytics-placement.md`

## 1. Purpose

ks-signal is the **analytics tier** between it2ks (keystroke capture) and trixi/mnemosyne
(runtime). It turns raw keystroke events into a **psychomotor state descriptor** — mnemosyne's
situational fingerprint, Layer 1.

Primary goal: **richer retrieval**. The keystroke layer's unique contribution is
**affect-matched retrieval** — "surface what I was doing *last time I was in this state*
(scattered / fatigued / wired), regardless of topic." Semantics can't do this; operational
context only crudely approximates it.

This spec covers **Lab v1 only** — the discovery/calibration work that must precede any runtime
snapshot. Runtime (daemon, state file) is deliberately out of scope until a feature set earns
promotion.

## 2. Where ks-signal sits

```
it2ks (capture, macOS)        → ~/.it2ks/logs/YYYY-MM-DD.jsonl
   │ (later: linux2ks/win2ks, same schema)
   ▼
ks-signal (analytics + lab)   → findings + (eventually) featurespec/
   │ reads JSONL; uses transcripts + HRV as offline labels
   ▼
trixi (runtime)               → composes the fingerprint, owns signal integration
```

- ks-signal knows: JSONL schema, keystroke features, baselines. It does **not** know nugs,
  mnemosyne, or trixi internals.
- **Integration of signals is trixi's job, not ks-signal's.** ks-signal is one sensor.
- The keystroke baseline must be built from the **full** keystroke stream — only ks-signal sees
  it (a nug-write-only sample would be biased). So the baseline lives here by necessity.

## 3. Input contract (from it2ks)

Newline-delimited JSON under `~/.it2ks/logs/YYYY-MM-DD.jsonl`.

- **Session record:** `type=session`, fields `s` (short index), `sid` (UUID), `app` (foreground
  process), `t0` (RFC3339 µs UTC).
- **Keystroke record:** `ts` (ns monotonic since process start), `wall` (RFC3339 µs UTC), `s`
  (session index), `act` (`down`/`up`/`flags`), `key` (keycode int), `char` (printable or empty).

Notes the reader must handle: dwell = down→matching up; expect fewer ups than downs (auto-repeat,
modifier-only `flags`); `ts` resets per it2ks process start, so **use `wall` for cross-session
alignment**, `ts` only for fine intra-session intervals.

## 4. Candidate feature set v1

Extract per rolling window (window size is a lab parameter, start ~30–60s of activity):

- **IKI** (inter-key-down interval): mean, variance/CV, median, p95.
- **Pause structure**: count of gaps >500ms, >2s; longest pause.
- **Burstiness**: ratio of fast (<150ms) runs to total.
- **Dwell** (down→up): mean, variance (auto-repeat-aware).
- **Error/correction**: backspace/delete rate per char.
- **Throughput**: keys/active-minute.

These are *candidates* — the lab keeps only those that predict a label (§6). All conditioned on
context where data allows: `(app, time-of-day)`.

## 5. Label sources (offline, lab-only)

Two opportunistic, high-quality ground-truth sources. Both rare; neither is a runtime signal.

- **Transcripts** — `~/.claude/projects/<proj>/<session>.jsonl`. Yields: modality
  (typed vs dictated, by keystroke presence/absence — spike-proven, F2), task context, and
  language sentiment. Coupling to Claude's undocumented format is acceptable **here only**.
- **HRV** — Polar, worn *sometimes*. Yields arousal ground truth during overlap windows.
  Export path TBD in the plan (Polar Flow export / a HealthKit bridge).

**Guardrail:** labels are lab inputs for feature discovery. They are **forbidden** in the runtime
snapshot path and in `featurespec/`.

**Granularity — window-level, not content-level (firm line).** The transcript joins to keystroke
*windows* by wall-clock and yields window-level labels (modality / task / sentiment). ks-signal
does **not** align keystrokes to individual words. Content-level alignment ("which word did I
pause on") is rejected: it's lossy (edits/paste/dictation break it), it turns a content-*blind*
psychomotor sensor into a psycholinguistic content analyzer (privacy line), and retrieval indexes
aggregate state, not per-word events. **Parked candidate:** *turn-taking timing* (latency-to-type,
compose duration, burst-vs-stall across a turn) uses message *boundaries*, not content — cheap and
privacy-clean, but conversational-context-only, so a candidate feature, not a v1 fundamental.

## 6. Calibration method (the core of v1)

The discovery loop that makes "lab-first" concrete:

1. Extract candidate features (§4) over a span of logs.
2. Align feature windows with labels (§5) during overlap windows (wall-clock join).
3. Test which features track which labels (e.g. does IKI-variance rise with low HRV / negative
   sentiment?). Simple correlation/effect-size first; nothing fancy.
4. **Keep** features that predict; **drop** the rest.
5. Survivors define the **baseline model** (per-subject, per-context distributions) and the
   eventual `featurespec` surface.

Output of v1 is a **finding** documenting at least one feature that demonstrably tracks at least
one label — proving the whole pipeline end-to-end.

## 7. Subject & input hygiene (minimal in v1)

- **Non-human-input filter** (cheap heuristic): reject windows with sub-physiological timing
  (sustained <20ms gaps, no plausible dwell distribution, machine regularity) — paste, auto-repeat,
  scripted input. (Note F1: STT/dictation does **not** enter the keystroke stream, so it needs no
  filter here.)
- **Single-subject presumption** in v1: dk is the dominant typist during the calibration span;
  treat all human-typing windows as the primary. Full primary-verification and per-subject
  baselines are designed but only enforced once a baseline exists (cold-start by phasing).
- **Deferred:** multi-profile enrollment, guest discrimination, cluster-discovery.

## 8. Outputs

- `docs/findings/` — dated write-ups of what the lab learned (which features track which labels).
- (Later, gated) `featurespec/` — the stable, versioned surface trixi may reference for snapshot
  schema. **Not** built in v1.
- (Later, gated) runtime snapshot `~/.ks-signal/state/current.json` carrying raw features +
  deviation + subject block. Daemon-vs-on-demand decided then, not now.

## 9. Testing

- **Reader**: golden-file tests over a captured JSONL sample (sessions, keystrokes, malformed
  lines, missing-up cases).
- **Feature extractors**: unit tests with hand-constructed event sequences and known expected
  values (IKI, dwell, pause counts).
- **Label alignment**: tests for wall-clock join correctness across session boundaries and clock
  edge cases.
- **Non-human filter**: synthetic injection bursts vs human-like sequences.

## 10. Non-goals (v1)

- No runtime daemon, no state file, no `featurespec` package.
- No signal integration (trixi's job), no runtime transcript correlation.
- No voice/HRV/focus *sensors* (HRV enters only as an offline label export).
- No multi-profile identity.

## 11. Open questions (for the plan)

- HRV export mechanism from Polar (Flow export vs HealthKit bridge) and its time resolution.
- Rolling-window size + step (lab parameter; sweep empirically).
- Language-sentiment labeling: heuristic vs a small local model over transcript text.
- Go module layout (reader / features / labels / findings) — defer to the plan.
