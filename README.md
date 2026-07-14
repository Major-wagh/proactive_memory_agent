# Proactive Memory Agent

Implementation workspace for the paper **"Remember When It Matters: Proactive Memory Agent for Long-Horizon Agents"** — Wu et al., Meta AI, July 2026 (arXiv:2607.08716).

**The idea in three sentences.** Long-horizon LLM agents fail not only by *losing* information, but by *ceasing to be steered* by information they already produced — task requirements, environment facts, failed attempts, diagnoses, open subgoals. The paper names this **behavioral state decay** and treats memory as an **intervention policy**: a sidecar memory agent maintains a small structured memory bank and, at each memory step, explicitly decides to **inject one concise memory-grounded reminder into the action agent's next call, or stay silent**. That selectivity — not storage, retrieval, or summarization — is what moves benchmarks: **+8.3 pp** pass@1 on Terminal-Bench 2.0 and **+6.8 pp** on τ²-Bench for Claude Sonnet 4.5, with gains persisting for stronger action agents.

## How to read this repo (progressive disclosure)

Read only as deep as you need:

| Level | Time | Where |
|---|---|---|
| 0 — Elevator pitch | 1 min | this README |
| 1 — The whole idea | 10 min | [`docs/context/part_01_the_idea.md`](docs/context/part_01_the_idea.md) |
| 2 — Topic deep dives | 5–10 min each | [`docs/context/README.md`](docs/context/README.md) — index of parts 01–08 |
| 3 — Source of truth | — | [`docs/paper/`](docs/paper/) — links to the paper (arXiv 2607.08716) + authors' code |
| Spec — what *we* build | 10 min | [`specs/000_scope_and_decisions.md`](specs/000_scope_and_decisions.md) |

## Layout

```
proactive_memory_agent/
├── README.md                 ← you are here (level 0)
├── LICENSE                   ← MIT
├── docs/
│   ├── context/              ← the paper digested into self-contained parts 01–09 (levels 1–2)
│   └── paper/                ← pointers to the paper & authors' code (level 3, source of truth)
└── specs/                    ← spec-driven development: specs precede code
    └── 000_scope_and_decisions.md
```

## Method (spec-driven)

No implementation code lands before its spec. `specs/000` fixes scope and records the owner's decisions. Subsequent numbered specs (`001_memory_bank`, `002_two_phase_agent`, `003_harness_adapter`, …) will define each component's interfaces and acceptance criteria before that component is built. Everything a spec asserts about the paper links back to a `docs/context/` part, which links back to the PDF.

## Status

- [x] Paper read and distilled into `docs/context/` (2026-07-14)
- [x] Spec 000 — scope and owner decisions
- [x] Authors' reference implementation vetted — verbatim prompts & injection format captured in [part 09](docs/context/part_09_authors_reference_implementation.md)
- [ ] Spec 001+ — component specs
- [ ] Implementation

## License

MIT (see [`LICENSE`](LICENSE)). The authors' reference implementation is Apache-2.0; any code or prompts adapted from it will carry attribution (see part 09 §8).
