# Tooling De-Risk Memo — 2026-07-17

> **What this is:** the evidence base for the tooling decisions in [spec 000](../../specs/000_scope_and_decisions.md) (OD-2 harness, OD-3 models) and the hard requirements those decisions impose on specs 002/003. Produced by a multi-agent web-research pass (4 parallel research angles → adversarial verification of each angle's load-bearing claim → synthesis). Every non-obvious claim is cited; §7 lists sources.
>
> **Bottom line:** the injection seam we need is **real and confirmed against primary docs** — but it lives in **LangChain v1 middleware** (`create_agent`), *not* in the deepagents feature stack. deepagents' own memory middleware injects into the system prompt on **every** call, which is the wrong shape for our transient one-call reminder. So **OD-2 changes from "deepagents" to "plain `create_agent` + one custom middleware, or a hand-rolled Ollama loop."** OD-3 (local ~4B open-weight) **holds**, gated on an on-device calibration harness, with a model fallback ladder. And the paper's trigger interval **N=1 is not runnable on 4 GB VRAM — default to N=4**.

## 1. Verdicts

| Decision | Verdict | One-line reason |
|---|---|---|
| **OD-2** — harness = deepagents | **ADJUST** | Seam exists but is inherited from LangChain `create_agent`; full deepagents stack is overkill and its Memory/Skills middleware inject persistently (wrong shape). Use `create_agent` + custom middleware, or a hand-rolled Ollama loop. |
| **OD-3** — models = local ~4B open-weight | **GO** | One shared `qwen3:4b` for both roles is feasible and avoids swap thrash — gated on a calibration harness + fallback ladder. |
| Trigger interval N=1 (paper) | **DEVIATE → N=4** | At ~15–30 tok/s a full extra memory pass every step ~doubles per-step latency; N=1 is a research-ideal, not runnable here. |

Provenance/confidence: the deepagents-seam and Ollama-feasibility findings were **adversarially re-verified against primary LangChain/Ollama docs** (`recommendation_holds: true`, high confidence). The dedicated "best 4B model" research agent **failed** (hit the structured-output retry cap after 5 tries) — so the `qwen3:4b` recommendation is **corroborated indirectly** by the Ollama-feasibility and harness-alternatives agents (both independently name Qwen3 as the most reliable small-scale tool-caller), not from a dedicated deep-dive. Treat the exact model pick as **provisional until the calibration harness runs** (§5).

## 2. The injection seam — confirmed, but it's LangChain, not deepagents

The paper's harness requirement is minimal: (a) read the recent message window before a model call; (b) attach a transient block to **just the next call** without touching the action agent's base prompt/tools/decoding. Both are met by **LangChain v1 agent middleware** (which `deepagents` sits on top of):

| Requirement | Mechanism | Notes |
|---|---|---|
| (a) observe history before a call | `before_model(state, runtime)` | runs before every model call with full access to the message list — read the sliding window here |
| (b) inject into next call only | `wrap_model_call(request, handler)` + `request.override(system_message=…)` | `override()` returns a **new immutable** `ModelRequest` scoped to that single call; base prompt/tools/state unchanged |
| custom hook attach point | `create_agent(..., middleware=[…])` | plain `create_agent`; `create_deep_agent` also accepts it but pulls in the rest of the stack |
| local models | `model="ollama:<name>"` | Ollama/vLLM/llama.cpp all supported; deepagents/LangChain are model-agnostic |

**Two implementation truths that MUST land in specs 002/003:**

1. **`wrap_model_call` fires on *every* model call — "only the next call" is *our* logic, not the hook's.** The middleware must read a **pending-reminder flag** from agent state, apply the `override`, then **clear the flag**. The single-call scope comes from our conditional + `override()`'s immutability, never from the hook alone. (This is also exactly how we get the paper's every-N-steps behavior.)
2. **Use `override(system_message=…)`** — the idiomatic transient path. `override(system_prompt=…)` is **deprecated**. `override(messages=…)` **replaces the entire list** (immutable new request), so to inject as a message you must pass `request.messages + [reminder]`, not just `[reminder]`.

**Why not the full deepagents stack:** its bundled `MemoryMiddleware`/`SkillsMiddleware` assemble their content into the **system message on every call** (persistent mutation) — the opposite of a transient one-call reminder — and its planning / virtual-filesystem / sub-agent machinery is dead weight that costs tokens+latency on a shared 4 B model. We'd be bypassing its headline feature anyway.

**Fidelity note:** the paper's authors did **not** use deepagents — they subclassed the Terminal-Bench/Terminus-2 harness and injected an XML-tagged block ([part 09](../context/part_09_authors_reference_implementation.md)). Terminus-2 is only worth adopting if we later want to reproduce their exact benchmark numbers.

## 3. The remaining fork (owner decision) — `create_agent`+middleware vs hand-rolled loop

Both cleanly give the seam and run on local Ollama. The memo rates them equally, differing on convenience vs. transparency:

- **Plain `create_agent` + one custom `AgentMiddleware`** — less plumbing to write (retry/streaming/state handled by LangChain); cost is pinning fast-moving `langchain`/`deepagents` 0.6.x versions and re-verifying `override()` semantics against the pinned build.
- **Hand-rolled ReAct/tool loop over Ollama `/api/chat`** — the seam is inherent (you own the outgoing message list), zero framework abstraction, and **4 B tool-call parse failures are directly observable** — which has real pedagogical value for a learning project. Cost: you write the tool-parse/dispatch loop yourself (small).

This is the one genuinely open decision left before spec 003.

## 4. Recommended model + Ollama configuration

- **Model:** `qwen3:4b` (Q4_K_M, ~2.4 GB on disk) — best-documented small-scale tool-caller, preferred over `gemma3:4b` specifically on tool-call reliability. **One shared instance** for action + memory agent.
- **Fallback ladder (gated on calibration):** `qwen3:4b` → `phi4-mini` (~3.8 B / ~3.2 GB) → `qwen3:1.7b`. Prevents an under-calibrated 4 B memory agent from silently degrading the action agent — the paper's warned failure.
- **Ollama env:**
  - `num_ctx=4096` (raise to 6–8K only if calibration shows headroom)
  - `OLLAMA_FLASH_ATTENTION=1` — **required**; without it KV-cache-type silently falls back to f16 → OOM. Verify `qwen3:4b` is on the flash-attention allowlist during calibration.
  - `OLLAMA_KV_CACHE_TYPE=q8_0` (fall back to `q4_0` if OOM)
  - `OLLAMA_NUM_PARALLEL=1` — avoids KV-cache multiplication; the two agents serialize in the loop anyway
  - long `keep_alive` to hold one resident model; confirm **"100% GPU"** via `ollama ps` — treat any CPU offload as a bug (spill collapses throughput ~5–20×)
- **Decoding:** `temperature=0` for all tool calls; constrain the four memory ops with a JSON schema (Ollama structured outputs) to push valid-call rate >90% at 4 B.

**Hardware reality:** 4 B-Q4 on a **4 GB laptop RTX 3050** is genuinely marginal — after WDDM/display reserve (~0.5–1 GB) the effective budget is ~3–3.5 GB, so KV-cache quantization is effectively mandatory to reach 4K context, and even 4K may spill. This is the dominant hardware risk.

## 5. New required gate — the calibration harness

Before N, `num_ctx`, and the model pick are locked, a small on-device harness must measure, on the actual RTX 3050:

1. **tool-call success rate** (target >90% valid calls for the four memory ops),
2. **tokens/sec** and realistic per-step latency under orchestration overhead,
3. **100% GPU residency** (`ollama ps`) at the chosen `num_ctx`.

Its results choose `qwen3:4b` vs the fallback ladder and confirm/adjust N and `num_ctx`. This becomes a milestone gate in spec 000 (M2.5, pre-eval).

## 6. Residual risks (ranked)

1. **4 B tool-call reliability** — the dominant, still-unverified risk and *exactly* the paper's "under-calibrated small memory model can hurt" failure. Small (<14 B) models widely report malformed JSON / missing args / calls to nonexistent paths. **Must be measured on-device.**
2. **4 GB headroom is optimistic** — display reserve + KV cache; the CPU-spill cliff (~40 → 2–5 tok/s) makes any offload catastrophic.
3. **Flash-attention allowlist** — not a global default in 2026; if `qwen3:4b` isn't on it, KV quant silently reverts to f16 → OOM. Verify, don't assume.
4. **Shared-model contention & N-deviation** — N=4 changes fidelity to the paper (log it); nested `wrap_model_call` latency under LangGraph overhead is unmeasured.
5. **LangChain/deepagents 0.6.x churn** — `override()` append-vs-replace, `system_prompt` deprecation, `ModelResponse` import churn. Pin versions; re-verify against the pinned build.
6. **Reproduction fidelity** — exact Terminal-Bench comparability is out of scope unless we adopt Terminus-2; our proof-of-effect eval (spec 004) is on our own decay-inducing tasks instead.

## 7. Sources

deepagents / LangChain middleware:
- https://docs.langchain.com/oss/python/langchain/middleware/custom
- https://reference.langchain.com/python/langchain/agents/middleware/types/ModelRequest
- https://reference.langchain.com/python/langchain/agents/middleware/types/ModelRequest/override
- https://docs.langchain.com/oss/python/deepagents/overview
- https://github.com/langchain-ai/deepagents
- https://www.langchain.com/blog/agent-middleware
- https://deepwiki.com/langchain-ai/deepagents/2.2-middleware-system
- https://reference.langchain.com/python/deepagents/middleware/memory
- https://pypi.org/project/deepagents/

Ollama tool-calling / VRAM / KV cache:
- https://docs.ollama.com/capabilities/tool-calling
- https://ollama.com/blog/streaming-tool · https://ollama.com/blog/tool-support
- https://docs.ollama.com/context-length · https://docs.ollama.com/faq
- https://github.com/ollama/ollama/issues/15043 · https://github.com/ollama/ollama/issues/13337
- https://github.com/ollama/ollama/pull/4943 · https://github.com/ollama/ollama/pull/3418
- https://mitjamartini.com/posts/ollama-kv-cache-quantization/
- https://smcleod.net/2024/12/bringing-k/v-context-quantisation-to-ollama/

Small-model tool-calling reliability / hardware:
- https://www.docker.com/blog/local-llm-tool-calling-a-practical-evaluation/
- https://localaimaster.com/blog/best-ollama-models-tool-calling
- https://localaimaster.com/blog/ollama-model-ram-vram-table
- https://www.promptquorum.com/prompt-bites/best-ollama-models-4gb-vram
- https://www.techreviewer.com/tech-specs/nvidia-rtx-3050-gpu-for-llms/
- https://willitrunai.com/gpus/rtx-3050-ti-laptop-4gb
- https://insiderllm.com/guides/local-ai-agents-guide/

Harness alternatives / reference:
- https://www.harborframework.com/docs/agents/terminus-2
- https://github.com/yifannnwu/proactive-memory-agent
- https://arxiv.org/abs/2607.08716

---
*Provenance: Workflow run `wf_c2851bbc-bc1` (8 agents; the `local-4b-model` research agent errored on the structured-output cap — model pick corroborated via other dimensions, flagged provisional). Findings are a snapshot of July 2026; re-verify version-specific API details against the pinned build before coding.*
