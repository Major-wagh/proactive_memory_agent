# Spec 000 — Scope & Decisions

**Status:** v0.2 — scope decisions recorded 2026-07-14 (owner Q&A)
**Owner:** siddw143
**Grounding:** [`../docs/context/`](../docs/context/README.md) parts 01–08 (paper digest); build gaps G1–G11 in [part 08](../docs/context/part_08_limits_and_open_questions.md).

## 1. Mission

An **open-source implementation** of *"Remember When It Matters: Proactive Memory Agent for Long-Horizon Agents"* (Meta AI, arXiv:2607.08716) whose goal is to **demonstrate that memory-as-intervention works** — a research codebase, not a product. The owner will steer the codebase, and it **may deliberately deviate from the paper**; every deviation is appended to the Fidelity Ledger (§7) so "paper-faithful" vs. "ours" stays auditable.

**North-star claim to prove:** *adding the memory sidecar to an unmodified action agent improves long-horizon task success at acceptable token/latency cost — including with open-weight local models.*

Note this last clause goes **beyond the paper**: its prompted-memory evidence uses a frontier memory model (Claude Opus 4.6), and its own Table 4a warns that an *uncalibrated* 27B memory agent makes the agent worse (part 07 §6). Proving the effect with well-prompted local models is this project's potential novel contribution — and its main risk (R1).

## 2. Decisions (resolved 2026-07-14)

| ID | Decision | Choice | Notes |
|---|---|---|---|
| OD-1 | Scope | **Architecture + lightweight proof-of-effect eval** *(Claude's proposal on owner's request — amend freely)* | Reusable module + credible baseline-vs-memory A/B evidence. Not full Terminal-Bench/τ² reproduction; not training (yet). |
| OD-2 | Harness | **deepagents** (LangGraph-based, `langchain-ai/deepagents`) as the first adapter; **core stays harness-agnostic** | The sidecar attaches through deepagents/LangChain middleware hooks — observe recent messages before a model call + attach a transient context block to that call, which is exactly the paper's required seam (part 03 §2). A Claude Agent SDK adapter can come later as a second thin layer. |
| OD-3 | Models | **Open-weight local models** via OpenAI-compatible endpoints (Ollama / vLLM); Qwen3-class default | Exact sizes pending owner hardware (OI-2). The **memory role gets the strongest available model** — the paper's asymmetry (Opus memory over Sonnet actor) suggests memory quality matters most. |
| OD-4 | Memory scope | **Per-task bank only** (paper-faithful) | Bank created at task start, serialized to JSON per run for inspection, discarded after. No cross-session store in v1. |

## 3. Fixed by the paper — normative requirements ("paper-faithful core")

| ID | Requirement | Source |
|---|---|---|
| FR-1 | Action agent is **unmodified**: no prompt surgery, tool or decoding changes; the only delta is an optional transient context block on its next call | part 03 |
| FR-2 | Memory agent runs at the first step then every N steps (default N=1, configurable), with inputs (task `x`, last-k message window (default k=8), bank `B`) | part 03 |
| FR-3 | Phase 1 emits an **ordered list** of tool calls from {`memory_update_status`, `memory_save_knowledge`, `memory_save_procedural`, `memory_delete`}; executed in order; empty list = bank unchanged; free-form bank rewriting is impossible by construction | part 04 |
| FR-4 | Bank = status (private, **never injected**) + knowledge + procedural; entries = {id, content, metadata(created_at, access stats)}; compact tagged content | part 04 |
| FR-5 | Phase 2 reads (x, w, B_t), **never edits the bank**, outputs exactly one of: one concise reminder, or explicit null (`<context_for_action>` xor `<no_intervention/>`) | part 05 |
| FR-6 | Reminders are memory-grounded (requirement / env fact / failed attempt / diagnosis / open subgoal); no strategy advice, no restating visible info, no plan takeover | part 05 |
| FR-7 | Injection is **transient**: next action call only; silence ⇒ next call byte-identical to baseline | part 03 §5 |
| FR-8 | Observability: bank serializable; every memory step logs Phase-1 calls, Phase-2 decision, tokens, latency; **intervention rate** reported per run | derived: parts 06 §6, 07 §7, 08 §5 |
| FR-9 | Trigger function pluggable: fixed-interval default; tool-error / failed-test triggers as extension points | part 03 §4 |

## 4. Defaults adopted from the paper

N = 1 (memory step every action step) · k = 8 (window) · ≤ 1 reminder per memory step · Phase-2 output tags `<context_for_action>` / `<no_intervention/>`.

## 5. Milestones & acceptance criteria

- **M1 — Core library** (pure Python, zero harness deps): `MemoryBank` + 4 tools + two-phase orchestrator + trigger + injection contract, tested against a scripted mock LLM.
  *Accept:* unit tests prove FR-3/4/5/7 semantics — ordered edits, legal no-op, status privacy, transient injection, silence ⇒ unchanged call.
- **M2 — deepagents adapter + demo**: middleware wrapping a deepagents agent; local models via Ollama/vLLM; one scripted long-horizon demo task.
  *Accept:* end-to-end run on local models; transcript shows ≥1 justified silence and ≥1 grounded intervention; logs complete per FR-8.
- **M3 — Proof-of-effect eval**: small reproducible suite of verifier-checked long-horizon tasks (candidates: SETA subset, terminal-style tasks, scripted τ²-like dialogues — OI-4), baseline vs. +memory, ≥3 seeds.
  *Accept:* report in README with pass-rate delta, token/latency overhead, intervention rate, per-seed variance.
- **M4 (stretch)** — selective-trigger A/B (paper names but never tests them); model-pairing matrix; much later, the training track (part 07).

## 6. Non-goals (v1) and risks

**Non-goals:** cross-session / persistent user memory · SFT/GRPO training · full benchmark reproduction · product concerns (multi-tenancy, UI).

**Risks:**
- **R1 — Local memory-model calibration.** Paper's strongest warning: an uncalibrated small memory agent *degrades* performance. Mitigation: strongest local model on the memory role; silence-biased prompting; watch intervention rate from the first run.
- **R2 — Prompts unpublished (G1).** ~~We write our own Phase-1/2 prompts.~~ **Resolved 2026-07-14:** the authors' repo publishes both system prompts (Apache-2.0) — see [part 09](../docs/context/part_09_authors_reference_implementation.md). We adapt them with attribution; risk downgraded to prompt-*tuning* for local models.
- **R3 — deepagents API drift.** Verify the current middleware API against live docs at implementation time (spec 003).
- **R4 — Eval credibility at small n.** Seeds + variance reporting; expect ±1–2 pp noise, as observed in the paper itself (part 06 §6).

## 7. Fidelity ledger (deviations from the paper — append as we steer)

| # | Deviation | Rationale | Status |
|---|---|---|---|
| D1 | Phase-2 reminders must cite bank entry IDs | Auditability; makes grounding measurable | planned |
| D2 | Open-weight models for **both** roles | OD-3; beyond the paper's prompted-frontier evidence | planned |
| D3 | Intervention rate + token/latency metering as first-class outputs | Paper gap G9; needed for the north-star claim | planned |

## 8. Open items (not blocking M1)

- **OI-1** ~~License + GitHub remote.~~ **Resolved 2026-07-14:** MIT license added (`LICENSE`); published to GitHub. Portions later adapted from the authors' Apache-2.0 repo will carry attribution via `THIRD_PARTY_NOTICES.md`.
- **OI-2** Owner hardware (GPU / VRAM) → concrete model sizes (e.g. Qwen3 8B / 14B / 30B-A3B via Ollama).
- **OI-3** ~~Vet the authors' repo.~~ **Resolved 2026-07-14:** vetted — findings in [part 09](../docs/context/part_09_authors_reference_implementation.md); gaps G1–G6, G10, G11 closed, G8 partial, G9 still ours.
- **OI-4** Task source for the M3 proof eval.

## 9. Next specs

`001_memory_bank.md` (schema, tools, serialization — closes G3/G4/G5/G11) · `002_two_phase_agent.md` (prompts, injection wire format, sync model — closes G1/G2/G6/G7/G8) · `003_deepagents_adapter.md` (middleware seam — closes G10) · `004_proof_eval.md` (closes G9, OI-4).
