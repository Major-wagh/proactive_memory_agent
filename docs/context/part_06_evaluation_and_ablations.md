# Part 06 — Evaluation & Ablations

> **Read this when:** you want the evidence — benchmark setup, main numbers, what each ablation isolates, and the caveats to keep in mind when reproducing.
>
> **TL;DR:** +8.3 pp (Sonnet 4.5) / +2.4 pp (Opus 4.6) pass@1 on Terminal-Bench 2.0; +6.8 / +2.5 pp on the τ²-Bench weighted average. Ablations: a bank without selective intervention, forced injection, advisor-without-bank, and Mem0 retrieval all trail the full system on macro average. Expect ±1–2 pp run noise when reproducing.

## 1. The two benchmarks and why they were chosen (§4.1)

**Terminal-Bench 2.0** (Merrill et al. 2026) — autonomous command-line execution: inspect files, run commands, edit code, debug failures, satisfy **hidden verifier tests**. Decay source: local debugging loops — forgotten requirements, repeated failed commands, overlooked observations, lost diagnoses. Emphasizes *environment grounding, debugging continuity, procedural memory*.

**τ²-Bench** (Yao et al. 2024; Barres et al. 2025) — conversational tool-use agents in service domains (airline / retail / telecom) with a **user simulator** and **domain policies**. Decay source: facts stated by the user, observed via tools, or implied by policy stop influencing decisions. Emphasizes *policy adherence, user-state tracking, conversation-level execution state*.

## 2. Setup

- **Terminal-Bench 2.0:** official Terminus-2-style harness; 89 tasks, results reported on the **85 paired tasks** where both baseline and memory runs produced valid evaluations (4 excluded for docker failures unrelated to agent behavior). Binary verifier per task; pass@1 = fraction of tasks whose final verifier passes.
- **τ²-Bench:** base split — airline 50, retail 114, telecom 114 (278 tasks per configuration). One sampled conversation per task; pass decided by the benchmark evaluator. Reported per-domain + task-weighted average.
- **Agents:** action ∈ {Claude Sonnet 4.5, Claude Opus 4.6}; memory agent = Claude Opus 4.6 everywhere. The action agent is byte-identical across baseline and memory runs.
- **Memory config:** invoked at the first step then **every step**; window **k = 8** messages; Phase 1 then Phase 2; reminders injected transiently.

## 3. Main results (Table 1, verbatim)

| Benchmark | Domain / split | Action model | n | Baseline | + Memory | Δ |
|---|---|---|---|---|---|---|
| Terminal-Bench 2.0 | full set | Sonnet 4.5 | 85 | 37.6% | 45.9% | +8.3 pp |
| Terminal-Bench 2.0 | full set | Opus 4.6 | 85 | 43.5% | 45.9% | +2.4 pp |
| τ²-Bench | airline | Sonnet 4.5 | 50 | 68.0% | 78.0% | +10.0 pp |
| τ²-Bench | retail | Sonnet 4.5 | 114 | 49.1% | 58.8% | +9.6 pp |
| τ²-Bench | telecom | Sonnet 4.5 | 114 | 55.3% | 57.9% | +2.6 pp |
| τ²-Bench | task-weighted avg. | Sonnet 4.5 | 278 | 55.0% | 61.8% | +6.8 pp |
| τ²-Bench | airline | Opus 4.6 | 50 | 76.0% | 76.0% | +0.0 pp |
| τ²-Bench | retail | Opus 4.6 | 114 | 64.9% | 69.3% | +4.4 pp |
| τ²-Bench | telecom | Opus 4.6 | 114 | 63.2% | 64.9% | +1.8 pp |
| τ²-Bench | task-weighted avg. | Opus 4.6 | 278 | 66.2% | 68.7% | +2.5 pp |

How the paper reads it:

- Gains are **larger for the weaker actor but do not disappear for the stronger one** — "the benefit is not merely compensating for limited capacity." Sonnet-with-memory reaches the Opus baseline on Terminal-Bench (45.9% vs 43.5%).
- Domain sensitivity — airline/retail large, telecom small — is "consistent with our framing of memory as a **domain-sensitive intervention policy** rather than a fixed summarization mechanism."

## 4. Ablations (Table 2 — τ², Sonnet 4.5 actor, Opus 4.6 memory)

Design: the architecture has two agentic capabilities — Phase 1 (bank management) and Phase 2 (selective intervention). Each ablation removes one at a time.

| Variant | Phase 1 (memory) | Phase 2 (intervention) | What it tests |
|---|---|---|---|
| Full memory agent (ours) | bank management | selective reminder / silence | — |
| Full-bank context | bank management | **expose entire bank every step** | Is passive visibility enough? |
| Always inject | bank management | **forced reminder every step** (no silence) | Is the no-op just an efficiency knob? |
| Injection-only (no bank) | **skipped** | selective guidance / silence | Does guidance need persistent grounding? (≈ advisor models) |
| Mem0 | Mem0 ADD | vector+BM25 top-10 retrieval | Retrieval vs. intervention |

Results (pass@1 %; macro = domains weighted equally, micro = task-weighted):

| Variant | Airline | Retail | Telecom | Macro | Micro |
|---|---|---|---|---|---|
| Sonnet 4.5 baseline | 68.0 | 49.1 | 55.3 | 57.5 | 55.0 |
| **+ Full memory agent (ours)** | **78.0** | 57.0 | 57.9 | **64.3** | 61.2 |
| + Full-bank context | 74.0 | 52.6 | 57.9 | 61.5 | 58.6 |
| + Always inject | 72.0 | 58.8 | 59.6 | 63.5 | **61.5** |
| + Injection-only (no bank) | 62.0 ↓ | 54.4 | **66.7** | 61.0 | 60.8 |
| + Mem0 | 68.0 = | 59.6 | 58.8 | 62.1 | 60.8 |

Interpretation (§4.3):

- **Full-bank context** trails the full system by 2.8 macro / 2.6 micro — "maintaining a memory bank is useful but exposing all remembered state is not enough; the memory agent must **select what is currently behaviorally relevant**."
- **Always inject** edges micro by +0.3 (within run variance) but loses macro (−0.8) and airline (−6.0) — selective **silence is not merely an efficiency optimization**; it matters where interruptions are costly.
- **Injection-only** helps telecom (66.7 — best in column!) but *hurts* airline below baseline (62.0 < 68.0) — ungrounded advisor-style guidance "is less stable without persistent memory."
- **Mem0** improves the average but leaves airline flat — "a general memory layer can return relevant records, but it does not explicitly model **when a memory should enter the loop** or how it should be phrased as a targeted reminder."
- Bottom line: "the most balanced gains come from combining maintained execution-state memory with a selective intervention policy."

Nuance worth remembering: per-domain optima differ (injection-only wins telecom outright). The full system wins on **balance across heterogeneous domains**, not on every column.

## 5. Qualitative analysis (§4.4)

- **Terminal-Bench — debugging continuity.** In `regex-log`, memory reactivates both a requirement and a diagnosis (regex violates the boundary condition; misses single-digit IPv4 octets). In `adaptive-rejection-sampler`, memory tracks repeated failed file edits and later surfaces an environment-specific workaround. Pattern: preserving "what was tried, what was observed, what failed, and what still constrains the solution."
- **τ²-Bench — policy & interaction state.** The Gold-vs-Regular compensation case (trust verified records over user claims); blocking an invalid basic-economy modification. Interventions cluster **immediately before state-changing tool calls**.
- Dominant pattern overall: "memory helps when it **reactivates a specific piece of execution state at the moment the action agent is about to ignore it**" — not generic summarization. Residual failures are calibration errors (part 05 §5), not storage failures.

## 6. Reproduction caveats (ours, not the paper's)

- **Single-run pass@1, no variance bars.** Table 1 vs. Table 2 report different numbers for the *same* Sonnet+memory config (retail 58.8 vs 57.0; micro 61.8 vs 61.2) — presumably separate runs. Treat **±1–2 pp as noise**.
- Printed Δs occasionally differ from raw subtraction by 0.1 pp (rounding); §4.2's text quotes "+9.7" for retail where Table 1 prints +9.6.
- Airline has n = 50, so each task is worth 2 pp — airline swings dominate macro movements.
- 85/89 Terminal-Bench tasks (docker exclusions assumed benign).
- **Token/latency overhead is never measured** — the memory condition adds a frontier-model call at (in these runs) *every* step. Our reproduction should meter cost from day one.

---

**Next:** [part 07 — making the policy cheap](part_07_training_open_weight_memory.md) · [part 08 — limits & gaps](part_08_limits_and_open_questions.md)
