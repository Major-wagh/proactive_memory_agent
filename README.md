# Proactive Memory Agent

Implementation workspace for the paper **"Remember When It Matters: Proactive Memory Agent for Long-Horizon Agents"** — Wu et al., Meta AI, July 2026 (arXiv:2607.08716).

> [!IMPORTANT]
> **This is an independent, unofficial learning project. We are not the authors of the paper and claim no ownership of it or the ideas it introduces.** All research credit belongs to Yifan Wu, Lizhu Zhang, Yuhang Zhou, Mingyi Wang, Bo Peng, Serena Li, Xiangjun Fan, and Zhuokai Zhao (Meta AI). This repository exists to *learn from* and *reproduce* their work, and is **not affiliated with, endorsed by, or connected to Meta AI or the authors.** The paper's method and results are theirs; only this re-implementation and its notes are ours. See [Attribution & credit](#attribution--credit).

**The idea in three sentences.** Long-horizon LLM agents fail not only by *losing* information, but by *ceasing to be steered* by information they already produced — task requirements, environment facts, failed attempts, diagnoses, open subgoals. The paper names this **behavioral state decay** and treats memory as an **intervention policy**: a sidecar memory agent maintains a small structured memory bank and, at each memory step, explicitly decides to **inject one concise memory-grounded reminder into the action agent's next call, or stay silent**. That selectivity — not storage, retrieval, or summarization — is what moves benchmarks: **+8.3 pp** pass@1 on Terminal-Bench 2.0 and **+6.8 pp** on τ²-Bench for Claude Sonnet 4.5, with gains persisting for stronger action agents.

## How to read this repo (progressive disclosure)

Read only as deep as you need:

| Level | Time | Where |
|---|---|---|
| 0 — Elevator pitch | 1 min | this README |
| 1 — The whole idea | 10 min | [`docs/context/part_01_the_idea.md`](docs/context/part_01_the_idea.md) |
| 2 — Topic deep dives | 5–10 min each | [`docs/context/README.md`](docs/context/README.md) — index of parts 01–09 |
| 3 — Source of truth | — | [`docs/paper/`](docs/paper/) — links to the paper (arXiv 2607.08716) + authors' code |
| Spec — what *we* build | 10 min | [`specs/000_scope_and_decisions.md`](specs/000_scope_and_decisions.md) |

## Layout

```
proactive_memory_agent/
├── README.md                 ← you are here (level 0)
├── LICENSE                   ← MIT
├── docs/
│   ├── context/              ← the paper digested into self-contained parts 01–09 (levels 1–2)
│   ├── research/             ← tooling de-risk memos that inform the specs (evidence, cited)
│   └── paper/                ← pointers to the paper & authors' code (level 3, source of truth)
└── specs/                    ← spec-driven development: specs precede code
    ├── 000_scope_and_decisions.md
    ├── 001_memory_bank.md
    ├── 002_two_phase_agent.md
    └── 003_harness_adapter.md
```

## Method (spec-driven)

No implementation code lands before its spec. `specs/000` fixes scope and records the owner's decisions. Subsequent numbered specs (`001_memory_bank`, `002_two_phase_agent`, `003_harness_adapter`, …) will define each component's interfaces and acceptance criteria before that component is built. Everything a spec asserts about the paper links back to a `docs/context/` part, which links back to the PDF.

## Status

- [x] Paper read and distilled into `docs/context/` (2026-07-14)
- [x] Spec 000 — scope and owner decisions
- [x] Authors' reference implementation vetted — verbatim prompts & injection format captured in [part 09](docs/context/part_09_authors_reference_implementation.md)
- [x] Spec 001 — memory bank & Phase-1 tool surface ([specs/001](specs/001_memory_bank.md))
- [x] Tooling de-risked — injection seam + local-model/Ollama plan verified ([research memo](docs/research/2026-07-17_tooling_derisk.md)); harness = LangChain `create_agent` + custom middleware, model `qwen3:4b`, trigger N=4
- [x] Spec 002 — two-phase memory agent ([specs/002](specs/002_two_phase_agent.md))
- [x] Spec 003 — harness adapter (LangChain `create_agent` middleware) ([specs/003](specs/003_harness_adapter.md))
- [ ] Spec 004 — proof-of-effect eval (needs task-source decision, OI-4)
- [ ] Implementation (starts at M1 after spec review)

## Attribution & credit

**The paper, its method, and its results are the work of its authors — not us.** This repository is a third-party, educational re-implementation by a learner; nothing here should be read as an official artifact of the paper.

- **Paper:** *Remember When It Matters: Proactive Memory Agent for Long-Horizon Agents* — Yifan Wu, Lizhu Zhang, Yuhang Zhou, Mingyi Wang, Bo Peng, Serena Li, Xiangjun Fan, Zhuokai Zhao (Meta AI), arXiv:[2607.08716](https://arxiv.org/abs/2607.08716), 2026.
- **Authors' official code (Apache-2.0):** <https://github.com/yifannnwu/proactive-memory-agent>
- **This repo is:** an unofficial study/reproduction. Not affiliated with, endorsed by, or reviewed by the authors or Meta AI. Any errors here are ours, not the paper's.

If you want the source of truth, read the paper and the authors' repository above. The digest in [`docs/context/`](docs/context/README.md) is our study notes; where it quotes the paper it is marked, and where it diverges from the paper that is logged in the spec's Fidelity Ledger.

```bibtex
@article{wu2026memoryagent,
  title   = {Remember When It Matters: Proactive Memory Agent for Long-Horizon Agents},
  author  = {Wu, Yifan and Zhang, Lizhu and Zhou, Yuhang and Wang, Mingyi and Peng, Bo and Li, Serena and Fan, Xiangjun and Zhao, Zhuokai},
  journal = {arXiv preprint arXiv:2607.08716},
  year    = {2026},
  url     = {https://arxiv.org/abs/2607.08716}
}
```

## License

MIT (see [`LICENSE`](LICENSE)) — covers **only our own code, specs, and notes** in this repository, not the paper or its ideas. The authors' reference implementation is Apache-2.0; any code or prompts adapted from it will carry attribution (see [part 09 §8](docs/context/part_09_authors_reference_implementation.md)). Quotations and figures from the paper remain the intellectual property of its authors and are used here for study and commentary.
