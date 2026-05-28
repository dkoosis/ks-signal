# ks-signal

**Keystroke analytics — the signal extracted from raw keystroke noise.**

status: **brainstorming** (no implementation yet)

ks-signal is the **analytics tier** between [it2ks](https://github.com/dkoosis/it2ks) (capture)
and trixi/mnemosyne (runtime). It turns raw, timestamped keystroke events into a *psychomotor
state descriptor* — mnemosyne's situational fingerprint, Layer 1.

```
it2ks (capture)  →  ks-signal (analytics)  →  trixi (runtime)
  JSONL logs          state descriptor          retrieval cue
```

It reads it2ks JSONL logs, establishes a personal baseline, and expresses current typing
behavior as deviation from that baseline — a signal mnemosyne can attach to a memory so
retrieval can match on *"what state was I in when I wrote this."*

## Goals, scope, and current design thinking

See [`.claude/rules/north-star.md`](.claude/rules/north-star.md). The placement rationale
(why a sibling repo to it2ks, not in-tree in trixi) lives in the trixi proposal
`docs/proposals/2026-05-28-keystroke-analytics-placement.md`.

Design is being firmed up before any code lands.
