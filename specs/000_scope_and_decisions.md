# Spec 000 — Scope & Decisions

**Status:** v0.3 — tooling de-risked 2026-07-17 (OD-2 adjusted, OD-3 confirmed); scope decisions 2026-07-14 (owner Q&A)
**Owner:** siddw143
**Grounding:** [`../docs/context/`](../docs/context/README.md) parts 01–09 (paper digest + authors' code); build gaps G1–G11 in [part 08](../docs/context/part_08_limits_and_open_questions.md); tooling evidence in [`../docs/research/2026-07-17_tooling_derisk.md`](../docs/research/2026-07-17_tooling_derisk.md).

## 1. Mission

An **open-source implementation** of *"Remember When It Matters: Proactive Memory Agent for Long-Horizon Agents"* (Meta AI, arXiv:2607.08716) whose goal is to **demonstrate that memory-as-intervention works** — a research codebase, not a product. The owner will steer the codebase, and it **may deliberately deviate from the paper**; every deviation is appended to the Fidelity Ledger (§7) so "paper-faithful" vs. "ours" stays auditable.

**North-star claim to prove:** *adding the memory sidecar to an unmodified action agent improves long-horizon task success at acceptable token/latency cost — including with open-weight local models.*

Note this last clause goes **beyond the paper**: its prompted-memory evidence uses a frontier memory model (Claude Opus 4.6), and its own Table 4a warns that an *uncalibrated* 27B memory agent makes the agent worse (part 07 §6). Proving the effect with well-prompted local models is this project's potential novel contribution — and its main risk (R1).

## 2. Decisions (resolved 2026-07-14)

| ID | Decision | Choice | Notes |
|---|---|---|---|
| OD-1 | Scope | **Architecture + lightweight proof-of-effect eval** *(Claude's proposal on owner's request — amend freely)* | Reusable module + credible baseline-vs-memory A/B evidence. Not full Terminal-Bench/τ² reproduction; not training (yet). |
| OD-2 | Harness | **ADJUSTED 2026-07-17** → **LangChain v1 `create_agent` + one custom middleware**, *not* the full deepagents stack; **core stays harness-agnostic** | De-risk research ([memo](../docs/research/2026-07-17_tooling_derisk.md)) confirmed the seam (`before_model` to observe the window + `wrap_model_call`/`request.override(system_message=…)` to inject into just the next call) but found it lives in **LangChain middleware beneath deepagents** — deepagents' own Memory/Skills middleware inject into the system prompt on *every* call (wrong shape) and the rest of its stack is dead weight on a shared 4 B model. **Sub-decision resolved 2026-07-17: `create_agent` + one custom `AgentMiddleware`** (owner pick — stay in the LangChain ecosystem / use a maintained harness, minus the deepagents stack). Hand-rolled Ollama loop kept as a documented fallback only. |
| OD-3 | Models | **CONFIRMED (go) 2026-07-17** — **open-weight local via Ollama**; default **one shared `qwen3:4b` (Q4_K_M)** for both roles | Best-documented small-scale tool-caller (over `gemma3:4b`), directly targeting the paper's "under-calibrated small memory model hurts" warning. Shared instance avoids swap thrash. **Gated on a calibration harness (M2.5)** + fallback ladder `qwen3:4b → phi4-mini → qwen3:1.7b`. Ollama config (num_ctx=4096, `OLLAMA_FLASH_ATTENTION=1`, KV `q8_0`, `NUM_PARALLEL=1`, 100% GPU) in memo §4. |
| OD-4 | Memory scope | **Per-task bank only** (paper-faithful) | Bank created at task start, serialized to JSON per run for inspection, discarded after. No cross-session store in v1. |

## 3. Fixed by the paper — normative requirements ("paper-faithful core")

| ID | Requirement | Source |
|---|---|---|
| FR-1 | Action agent is **unmodified**: no prompt surgery, tool or decoding changes; the only delta is an optional transient context block on its next call | part 03 |
| FR-2 | Memory agent runs at the first step then every N steps (N **configurable**; paper reference N=1, **deployment default N=4** on 4 GB VRAM — see §4 and D4), with inputs (task `x`, last-k message window (default k=8), bank `B`) | part 03 |
| FR-3 | Phase 1 emits an **ordered list** of tool calls from {`memory_update_status`, `memory_save_knowledge`, `memory_save_procedural`, `memory_delete`}; executed in order; empty list = bank unchanged; free-form bank rewriting is impossible by construction | part 04 |
| FR-4 | Bank = status (private, **never injected**) + knowledge + procedural; entries = {id, content, metadata(created_at, access stats)}; compact tagged content | part 04 |
| FR-5 | Phase 2 reads (x, w, B_t), **never edits the bank**, outputs exactly one of: one concise reminder, or explicit null (`<context_for_action>` xor `<no_intervention/>`) | part 05 |
| FR-6 | Reminders are memory-grounded (requirement / env fact / failed attempt / diagnosis / open subgoal); no strategy advice, no restating visible info, no plan takeover | part 05 |
| FR-7 | Injection is **transient**: next action call only; silence ⇒ next call byte-identical to baseline | part 03 §5 |
| FR-8 | Observability: bank serializable; every memory step logs Phase-1 calls, Phase-2 decision, tokens, latency; **intervention rate** reported per run | derived: parts 06 §6, 07 §7, 08 §5 |
| FR-9 | Trigger function pluggable: fixed-interval default; tool-error / failed-test triggers as extension points | part 03 §4 |

## 4. Defaults

**Paper reference:** N = 1 (memory step every action step) · k = 8 (window) · ≤ 1 reminder per memory step · Phase-2 output tags `<context_for_action>` / `<no_intervention/>`.

**Deployment defaults (this hardware, from the 2026-07-17 de-risk memo):** N = **4** (N=1 ~doubles per-step latency at ~15–30 tok/s — impractical, logged as D4) · k = 8 (must fit the shared `num_ctx=4096` budget) · ≤ 1 concise reminder, injected via `request.override(system_message=…)` · `temperature=0` + JSON-schema-constrained tool calls. All configurable; the calibration harness (M2.5) confirms/adjusts N and `num_ctx`.

## 5. Milestones & acceptance criteria

- **M1 — Core library** (pure Python, zero harness deps): `MemoryBank` + 4 tools + two-phase orchestrator + trigger + injection contract, tested against a scripted mock LLM.
  *Accept:* unit tests prove FR-3/4/5/7 semantics — ordered edits, legal no-op, status privacy, transient injection, silence ⇒ unchanged call.
- **M2 — Harness adapter + demo**: the memory sidecar as a `create_agent` custom middleware (or hand-rolled Ollama loop — OD-2 sub-decision); `qwen3:4b` via Ollama; one scripted long-horizon demo task.
  *Accept:* end-to-end run on local models; transcript shows ≥1 justified silence and ≥1 grounded intervention; action agent's base prompt/tools provably untouched; logs complete per FR-8.
- **M2.5 — Calibration harness (gate)**: on-device measurement on the RTX 3050 — tool-call valid-rate (target >90% for the four memory ops), tokens/sec + per-step latency, 100% GPU residency (`ollama ps`) at `num_ctx=4096`.
  *Accept:* locks the model pick (`qwen3:4b` vs fallback ladder) and the N / `num_ctx` defaults with real numbers before the eval. If 4 B misses the tool-call bar, drop to `phi4-mini`/`qwen3:1.7b` per the ladder.
- **M3 — Proof-of-effect eval**: small reproducible suite of verifier-checked long-horizon tasks, **deliberately decay-inducing** so a 4 B memory agent's effect is detectable (candidates: SETA subset, terminal-style tasks, scripted τ²-like dialogues, synthetic fact-then-filter tasks — OI-4), baseline vs. +memory, ≥3 seeds.
  *Accept:* report in README with pass-rate delta, token/latency overhead, intervention rate, per-seed variance.
- **M4 (stretch)** — selective-trigger A/B (paper names but never tests them); model-pairing matrix; much later, the training track (part 07).

## 6. Non-goals (v1) and risks

**Non-goals:** cross-session / persistent user memory · SFT/GRPO training · full benchmark reproduction · product concerns (multi-tenancy, UI).

**Risks:**
- **R1 — Local memory-model calibration.** Paper's strongest warning: an uncalibrated small memory agent *degrades* performance. Mitigation: strongest local model on the memory role; silence-biased prompting; watch intervention rate from the first run. **Sharpened by OI-2 (4 GB VRAM):** a ~4B memory policy is far below the paper's Opus sidecar, so the proof-of-effect eval (spec 004) should include tasks that *deliberately induce* behavioral state decay (long filler between fact and use) so the effect is detectable at this scale. Escape hatch if the local proof stalls: the model client is any OpenAI-compatible endpoint, so an API memory model is a config-only diagnostic — owner decides if/when. **Now front-loaded as a hard gate:** the M2.5 calibration harness measures the memory agent's on-device tool-call valid-rate *before* the eval, so a mis-calibrated 4 B model is caught (and swapped down the fallback ladder) rather than silently hurting results.
- **R2 — Prompts unpublished (G1).** ~~We write our own Phase-1/2 prompts.~~ **Resolved 2026-07-14:** the authors' repo publishes both system prompts (Apache-2.0) — see [part 09](../docs/context/part_09_authors_reference_implementation.md). We adapt them with attribution; risk downgraded to prompt-*tuning* for local models.
- **R3 — LangChain/deepagents API churn.** ~~Verify middleware API.~~ **De-risked 2026-07-17:** seam confirmed against primary docs (`create_agent` + `wrap_model_call`/`override`), and we dropped to plain `create_agent` to shed the deepagents stack. Residual: 0.6.x churn (`override()` append-vs-replace, `system_prompt` deprecation, `ModelResponse` import). Mitigation: pin `langchain`/`langgraph` versions, re-verify `override()` semantics against the pinned build before coding (spec 003). A hand-rolled Ollama loop removes this risk entirely if we take the OD-2 sub-decision that way.
- **R5 — 4 GB VRAM / flash-attention footguns (new).** 4 B-Q4 on a 4 GB laptop card is marginal (~3–3.5 GB effective budget); KV-cache quant is effectively mandatory and requires `OLLAMA_FLASH_ATTENTION=1` (else silent f16 fallback → OOM). CPU spill collapses throughput ~5–20×. Mitigation: M2.5 calibration harness verifies 100% GPU residency and FA-allowlist membership before anything is locked.
- **R4 — Eval credibility at small n.** Seeds + variance reporting; expect ±1–2 pp noise, as observed in the paper itself (part 06 §6).

## 7. Fidelity ledger (deviations from the paper — append as we steer)

| # | Deviation | Rationale | Status |
|---|---|---|---|
| D1 | Phase-2 reminders must cite bank entry IDs | Auditability; makes grounding measurable | planned |
| D2 | Open-weight models for **both** roles | OD-3; beyond the paper's prompted-frontier evidence | planned |
| D3 | Intervention rate + token/latency metering as first-class outputs | Paper gap G9; needed for the north-star claim | planned |
| D4 | Trigger interval **default N=4** (paper uses N=1) | N=1 ~doubles per-step latency at ~15–30 tok/s on 4 GB VRAM; N configurable, N=1 available for reference runs | accepted 2026-07-17 |
| D5 | Harness is `create_agent`+middleware / hand-rolled loop, not the authors' Terminus-2 subclass | OD-2; local-first, plug-and-play; exact Terminal-Bench reproduction is out of scope (part 06) | accepted 2026-07-17 |

## 8. Open items (not blocking M1)

- **OI-1** ~~License + GitHub remote.~~ **Resolved 2026-07-14:** MIT license added (`LICENSE`); published to GitHub. Portions later adapted from the authors' Apache-2.0 repo will carry attribution via `THIRD_PARTY_NOTICES.md`.
- **OI-2** ~~Owner hardware → model sizes.~~ **Resolved 2026-07-14:** NVIDIA RTX 3050, **4 GB VRAM**, Windows 11. Runtime = **Ollama** (native Windows; vLLM needs WSL2 — deferred). Default v1 pairing: **one shared model for both roles** — Qwen3-4B-class at Q4 quantization (~2.5 GB weights; roughly 8K context realistic with KV-cache quantization — verify empirically at M2, and check whether a newer ~4B Qwen has been released by then). A single resident model avoids per-step model-swap thrash on one GPU. Alternates to test at M2: Qwen3-1.7B action + 4B memory (memory role always gets the stronger model); 8B-class only via CPU offload — likely too slow at trigger interval N=1, plausible at N≥4. Consequences: trigger interval N and context char budgets become the primary knobs (spec 002); same-model-both-roles is paper-consistent (the Opus actor + Opus memory row still gained +2.4/+2.5 pp).
- **OI-3** ~~Vet the authors' repo.~~ **Resolved 2026-07-14:** vetted — findings in [part 09](../docs/context/part_09_authors_reference_implementation.md); gaps G1–G6, G10, G11 closed, G8 partial, G9 still ours.
- **OI-4** Task source for the M3 proof eval.

## 9. Next specs

[`001_memory_bank.md`](001_memory_bank.md) (schema, tools, serialization — closes G3/G4/G5/G11) — **written 2026-07-14** · `002_two_phase_agent.md` (prompts, `override(system_message=…)` injection contract, sync model, shared-`num_ctx` budgets, N=4 — closes G1/G2/G6/G7/G8) · `003_harness_adapter.md` (`create_agent`+middleware **or** hand-rolled Ollama loop per OD-2 sub-decision; pending-reminder gating; version pins — closes G10) · `004_proof_eval.md` (decay-inducing tasks, closes G9, OI-4). Specs 002/003 build directly on the [2026-07-17 de-risk memo](../docs/research/2026-07-17_tooling_derisk.md).
