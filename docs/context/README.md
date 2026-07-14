# Paper Context — Index

Digest of **"Remember When It Matters: Proactive Memory Agent for Long-Horizon Agents"** (Wu et al., Meta AI, arXiv:2607.08716, July 10, 2026), split into self-contained parts. Every part opens with *Read this when* + a TL;DR so you can stop as early as possible (progressive disclosure). The original PDF and a raw text extraction (with minor extraction artifacts, e.g. missing spaces) live in [`../paper/`](../paper/).

| # | Part | One-liner |
|---|------|-----------|
| 01 | [The idea](part_01_the_idea.md) | The whole paper in ten minutes: what it is, who it's for, problem, solution, results, vocabulary. |
| 02 | [Behavioral state decay](part_02_behavioral_state_decay.md) | The failure mode, why bigger contexts / RAG / summarizers don't fix it, positioning vs. adjacent work. |
| 03 | [Architecture & control loop](part_03_architecture_and_control_loop.md) | Formal setup, sidecar integration, triggering, transient-injection semantics. |
| 04 | [Memory bank & tools](part_04_memory_bank_and_tools.md) | The status / knowledge / procedural bank and the four Phase-1 edit tools. |
| 05 | [Intervention policy](part_05_intervention_policy.md) | Phase 2: when to speak, when to stay silent, observed failure modes. |
| 06 | [Evaluation & ablations](part_06_evaluation_and_ablations.md) | Benchmarks, main numbers, what each ablation isolates, reproduction caveats. |
| 07 | [Training open-weight memory](part_07_training_open_weight_memory.md) | SFT + GRPO recipe on SETA, pivot turns, transfer results. |
| 08 | [Limits & open questions](part_08_limits_and_open_questions.md) | What the paper leaves unspecified — the direct input to our specs. |
| 09 | [Authors' reference implementation](part_09_authors_reference_implementation.md) | Vetted authors' repo: the real Phase-1/2 prompts, injection wire format, bank internals — closes most gaps from part 08. |

## Reading paths

- **"Just tell me the idea"** → 01.
- **Implementing the module** → 03 → 04 → 05 → **09**, then [`../../specs/`](../../specs/).
- **Reproducing the results** → 06 (plus 03 §6 for configuration defaults).
- **Training a memory model** → 07.
- **Reviewing / de-risking the project** → 02 → 08.

## Conventions

- Quoted text is verbatim from the paper; section references (e.g. §3.2) point into the PDF.
- **Paper gap:** callouts mark details the paper does *not* specify. They are collected in part 08 and resolved in the specs.
- Anything labelled *illustrative reconstruction* is our example of a format the paper describes but never prints literally.
