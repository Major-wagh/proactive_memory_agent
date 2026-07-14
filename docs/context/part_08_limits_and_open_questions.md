# Part 08 — Limits, Gaps & Open Questions

> **Read this when:** de-risking the project, or turning the paper into a build spec. This part collects (a) the authors' stated open directions, (b) engineering details the paper does **not** specify, (c) threats to validity, and (d) the decisions we must make ourselves — which are resolved in [`../../specs/000_scope_and_decisions.md`](../../specs/000_scope_and_decisions.md).

## 1. The authors' stated open directions (§5)

1. **Jointly training** the memory and action agents (the paper trains memory only, actor frozen).
2. **Learning when memory is invoked** (learned triggers) instead of a fixed schedule.
3. Identifying when **verbatim reminders** vs. **task-specific abstractions** are most effective.

## 2. What the paper leaves unspecified (build gaps)

These are the **Paper gap** callouts from parts 03–07, consolidated. Each becomes a decision in a numbered spec.

| # | Gap | Why it matters | Where we decide |
|---|---|---|---|
| G1 | Exact Phase-1 / Phase-2 prompts | The core "IP" of a prompted memory agent; quality of the whole system | spec 002 (check authors' repo first) |
| G2 | Injection wire format — role, wrapper, position in prompt (only the tags `<context_for_action>` / `<no_intervention/>` are given) | Materially affects actor behavior | spec 002 |
| G3 | Tag vocabulary & entry format details | Bank legibility and Phase-2 grounding | spec 001 |
| G4 | Bank growth: size limits, eviction, dedup | Long tasks could bloat Phase-1 context | spec 001 |
| G5 | Access statistics: recorded, but usage never described | Possible relevance signal; unclear semantics | spec 001 (record, don't consume in v1) |
| G6 | Sync vs. async memory step; latency accounting | Throughput of real deployments; "separate process" framing vs. gating the next call | spec 002 |
| G7 | Interval N and window k beyond defaults (N=1, k=8) | The main cost knob | spec 002 (configurable) |
| G8 | Behavior when a reminder is wrong or stale | Actor cannot contest; wrong reminders observed to cause extra verification | spec 002 (log; consider feedback into bank) |
| G9 | Token / latency overhead | Never measured in the paper | our eval must meter it |
| G10 | Harness specifics — how injection coexists with the harness's own truncation/summarization | Needed to reproduce benchmark numbers | spec 003 (if we reproduce evals) |
| G11 | Entry update semantics (no in-place edit tool) | Presumably delete + re-save | spec 001 |

## 3. Threats to validity (read the numbers with these in mind)

- **Single-run pass@1, no error bars.** Table 1 vs. Table 2 disagree on the identical Sonnet+memory config (retail 58.8 vs 57.0; micro 61.8 vs 61.2) → ~1–2 pp run noise. Airline n=50 makes each task worth 2 pp.
- **Memory model ≥ action model** in the headline rows (Opus 4.6 memory for Sonnet 4.5 actor). Opus+Opus still gains (+2.4 / +2.5 pp), so the effect isn't only "a bigger model helping a smaller one" — but the largest gains do use a stronger memory model than actor.
- **Both benchmarks are verifier-gradable domains.** No evidence yet on open-ended long-horizon work (research, writing, design) where "success" is not binary.
- **Training study is explicitly preliminary:** one model family (Qwen3.5), one domain (terminal tasks), small solved-count deltas, validation-set size unstated.
- 85/89 Terminal-Bench tasks (docker exclusions assumed unrelated to agent behavior).

## 4. Decisions we own (not the paper's to make)

Resolved with the project owner and recorded in [spec 000](../../specs/000_scope_and_decisions.md):

- **OD-1 Scope** — architecture only vs. + benchmark reproduction vs. + training track.
- **OD-2 Harness** — which agent loop the sidecar attaches to first.
- **OD-3 Models** — which models play actor and memory agent initially.
- **OD-4 Memory scope** — per-task bank (as in the paper) vs. adding cross-session persistence.

## 5. Suggestions beyond the paper (ours)

- **Selective triggers as a cheap cost lever.** The paper names tool-error / failed-test / repeated-command triggers but never tests them — an easy A/B once we have the fixed-interval baseline.
- **Meter tokens and wall-clock per condition from day one** (closes G9, and is the first question anyone will ask).
- **Log every Phase-1/Phase-2 decision as structured events** — free future SFT data (part 07 §7).
- **Require Phase-2 reminders to cite bank entry IDs** — makes interventions auditable and grounding measurable.
- **Track intervention rate** (share of memory steps that inject) — the paper never reports it, yet it is the key calibration statistic.
