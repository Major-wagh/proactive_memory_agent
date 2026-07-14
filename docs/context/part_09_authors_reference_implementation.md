# Part 09 — The Authors' Reference Implementation (vetted 2026-07-14)

> **Read this when:** writing specs 001–003. The authors' code answers most of the "Paper gap" callouts from parts 03–05 — including the **actual Phase-1/Phase-2 system prompts**.
>
> **TL;DR:** <https://github.com/yifannnwu/proactive-memory-agent> is public, **Apache-2.0**, and contains the full prompted memory agent ("Memory Agent V3"), the memory bank, the trigger, and a Terminus-2 subclass showing exactly how injection works. It uses `litellm` (so local vLLM/Ollama endpoints work out of the box — good for our OD-3). Our code stays MIT; anything we adapt from theirs keeps Apache-2.0 attribution (§8).

## 1. Repo facts

| Field | Value |
|---|---|
| URL | <https://github.com/yifannnwu/proactive-memory-agent> |
| License | Apache-2.0 (`external/harbor/` is a vendored fork of `laude-institute/harbor`) |
| LLM layer | `litellm` — "works with OpenRouter, OpenAI, Anthropic, local vLLM, etc." |
| Core layout | `src/memory_agent/memory/` (bank, trigger, prompts, agent) · `memory_enabled_agent.py` (Terminus-2 subclass) · `runner.py` / `cli.py` / `config.py` · `configs/*.yaml` |
| Extras | `examples/` — rendered baseline-vs-memory trajectories for 5 TB tasks (incl. `regex-log`, `adaptive-rejection-sampler`); full 89-task TB 2.0 task list in the configs |
| Naming note | Internally called "Memory Agent V3 — multi-turn tool use architecture". Some docstrings contradict each other ("single LLM call per step" vs. "two forward passes"); **the code does two LLM calls per memory step** (Phase 1 with tools, Phase 2 text-only). Trust the code, not the comments. |

## 2. The Phase-1 system prompt (verbatim)

From `src/memory_agent/memory/memory_agent.py` (`PHASE1_SYSTEM`), © the paper's authors, Apache-2.0:

```text
You are a Memory Bank Manager. Your job is to maintain a lean, accurate knowledge bank.

## Your Tools
- memory_save_knowledge: Save important FACTS (task requirements, env details, API constraints, file paths)
- memory_save_procedural: Record EXPERIENCES (failed approaches, error patterns, successful fixes)
- memory_update_status: Track progress internally (not shown to action agent)
- memory_delete: Remove outdated/incorrect entries

## Guidelines
1. **Be selective**: Only save information that would be LOST if the action agent's context window scrolls past it
2. **Prioritize task requirements**: API signatures, output formats, specific constraints from the task description
3. **Record failures**: What was tried, why it failed, what the error was — so it's not repeated
4. **Keep it lean**: Delete entries that are superseded by newer information
5. **Don't save obvious things**: Terminal output the agent just saw, general knowledge about tools

## What to save at Step 1 (task description):
- Specific API requirements (function signatures, argument orders, input types)
- Required output files and formats
- Constraints that are easy to forget mid-task
- Key domain-specific details

## What to save at later steps:
- Environment discoveries (package versions, tool availability)
- Error patterns and fixes
- Approach changes and why
- Performance or score tracking
```

## 3. The Phase-2 system prompt (verbatim)

`PHASE2_SYSTEM` — the "selective attention" framing is the heart of the method:

```text
You are a Selective Attention module for an Action Agent working on a coding task.

Your role is like selective attention — when the agent's working memory (context window)
has scrolled past important information, you can restore that context. But you also know
when things are fine and the agent doesn't need help.

## PROCESS:
1. Review the memory bank contents
2. Review the agent's recent trajectory (what it's doing now)
3. DECIDE: Does the agent need a context reminder right now?

## WHEN TO INTERVENE (output <context_for_action>):
- The agent has forgotten a key requirement or constraint stored in memory
- The agent is about to repeat a recorded failure pattern
- The agent's current approach contradicts a specific fact in the memory bank
- A format requirement, API detail, or constraint from the task seems forgotten

## WHEN NOT TO INTERVENE (output <no_intervention/>):
- The agent's actions are consistent with memory bank contents
- You're not confident that a specific fact is being forgotten
- The information you'd provide is already visible in the recent trajectory
- You'd only be restating what the agent already knows

## OUTPUT FORMAT:
If intervention needed, write:
<context_for_action>
Your synthesized context note here. You can:
- Highlight requirements that seem forgotten
- Note relevant past experiences from memory
- Connect dots between memory entries and current behavior
- Flag patterns that match recorded failures
Write as helpful observations, not commands.
</context_for_action>

If no intervention needed, write:
<no_intervention/>

## GUIDELINES:
- Your DEFAULT is <no_intervention/> — only intervene when you see a real gap
- When you do intervene, be thoughtful: synthesize, don't just dump memory entries
- Frame as observations: "The task requires X" not "You MUST do X"
- Don't include information already visible in the recent trajectory
- Be calibrated: don't pass things you're not sure about
- You CAN connect dots and highlight patterns — not limited to verbatim copies from bank
- The action agent is a strong model — help it remember, don't try to control it
```

Note it's already **silence-biased by design** ("Your DEFAULT is `<no_intervention/>`") — our spec's silence-biased-prompting idea is confirmed, not novel.

## 4. Bank implementation details (`universal_memory.py`)

- **Entry:** `{id, content, created_at, access_count, last_accessed}`. IDs are 8-char `shortuuid`s.
- **Tags are conventions, not schema:** knowledge `[ENV]`, `[PATH]`, `[FACT]`; procedural `[BUG]`, `[PERF]` (recommended in docstrings; the tool schemas take only a `content` string).
- **Status:** plain string, hard-capped at **2000 chars** (truncated with `...`); "~512 tokens, updated frequently".
- **Access statistics are deliberately disabled** — comment: only the BM25 path incremented them, "but context is now LLM-generated (Phase 2), making counts misleading."
- **No eviction.** Growth is handled prompt-side: if total entries > 50, the memory agent's own prompt shows only BM25 top-20 knowledge + top-20 procedural (filtered against the current observation).
- JSON `save_to_file` / `load_from_file` built in.
- **The status-exclusion war story** (comment in `render_context`): status is excluded from anything shown to the actor because *"Status values like 'Task completed successfully' cause the action agent to stop iterating before the task is truly finished."* The paper says status is private "to avoid polluting" — the code reveals the concrete failure that motivated it.

## 5. Two-phase agent mechanics (`memory_agent.py`)

- Phase 1: system=`PHASE1_SYSTEM`, tools=`BANK_TOOLS` → execute returned tool calls in order. Phase 2: system=`PHASE2_SYSTEM`, **no tools**, fresh prompt containing the **updated** bank → parse `<context_for_action>` / `<no_intervention/>`.
- Both phase prompts embed: step number, formatted `<memory_bank>` (status + `[id] content` lines), and the observation (task description + recent trajectory).
- **Robustness details worth copying:**
  - Unparseable Phase-2 output (no tags) → **default to no intervention** (fail-silent, never fail-noisy).
  - Fallback XML `<tool_call>` parser for SFT-trained (Qwen-style) models whose structured tool calls come back empty — directly relevant to our local-model plan.
  - `<think>…</think>` blocks stripped from responses (reasoning models).
  - `memory_delete` of an unknown ID is recorded as a failed op, not an exception.

## 6. Harness integration (`memory_enabled_agent.py`, a Terminus-2 subclass)

- **Synchronous sidecar (closes G6):** after each action step's observation is recorded, the memory agent is `await`ed *in the loop* — it gates the next action call; it is not actually a parallel process.
- **Memory-agent context (their `w_t`):** the **full task description is always included**, plus a sliding window of the last 8 steps, where each step entry = agent analysis + plan + first 5 commands + terminal output (already truncated to ~10 KB/step by Terminus-2). The window slides over a never-cleared accumulation list.
- **Injection wire format (closes G2):** a pending reminder is consumed **one-shot** and wrapped as:

  ```text
  <memory_context>
  The following context from memory may be relevant to your current work
  (these are observations, not directives — verify before acting):
  {reminder}
  </memory_context>
  ```

  Two placements via `injection_method`: `"system_prompt"` (class default — passed as the system arg of that single call) or `"user_turn"` (appended to the user prompt). **The Terminal-Bench config uses `"user_turn"`** — so the paper's headline numbers presumably used user-turn injection.
- **Built-in calibration stats:** trigger count, operation count, **injection count, no-op count** tracked per run and dumped to `trajectory_memory.json` (schema `memory-trajectory-v3`) together with both phase prompts, raw responses, and reasoning per memory step. Per-step token/cost metrics are recorded on the action side too. Our FR-8 is essentially their logging, kept.
- Coexists with the harness's own **proactive summarization** (`enable_summarize: true`) — memory injection and context summarization run together (closes G10).

## 7. Run configuration (TB 2.0 memory config)

| Knob | Value |
|---|---|
| Action model | `openrouter/anthropic/claude-sonnet-4.5`, temp 0.7, max 50 turns, XML parser |
| Memory model | `openrouter/anthropic/claude-opus-4.6`, temp **0.3** |
| Trigger | interval 1 + first step |
| Sliding window | 8 |
| Injection | `user_turn` |
| Episode resets | status: yes · knowledge: no · procedural: no |
| Memory-prompt cap | 50 entries, then BM25 top-20 + top-20 |

## 8. Gap closure & licensing

| Gap (part 08) | Status after vetting |
|---|---|
| G1 prompts | **Closed** — §2, §3 above |
| G2 injection format | **Closed** — §6 (`<memory_context>` wrapper, user_turn in TB) |
| G3 tag vocabulary | **Closed** — `[ENV] [PATH] [FACT] [BUG] [PERF]`, convention not schema |
| G4 bank growth | **Closed** — no eviction; BM25 top-20 prefilter beyond 50 entries |
| G5 access stats | **Closed** — fields exist, tracking deliberately disabled (misleading) |
| G6 sync vs async | **Closed** — synchronous, gates the next call |
| G7 N & k | Confirmed: N=1, k=8, exposed as config |
| G8 wrong reminders | **Partial** — wrapper hedges ("observations, not directives — verify before acting"); no bank feedback loop |
| G9 cost overhead | Still open — per-step tokens/cost logged but never reported |
| G10 harness coexistence | **Closed** — runs alongside Terminus-2 summarization |
| G11 entry update | **Closed** — delete + re-save; no in-place edit tool |

**Licensing:** their repo is **Apache-2.0**; our repo is MIT. Quoting their prompts here is attributed reference documentation. If we port or adapt their code/prompts into our source tree, those portions keep Apache-2.0 attribution (add a `THIRD_PARTY_NOTICES.md` at that point). Training/SETA code is **not** in the repo — only the prompted architecture and the TB runner.

## 9. What we still do differently (feeds specs 001–003)

- **deepagents middleware adapter** instead of a Terminus-2 subclass (OD-2) — same seam, different harness.
- **Reminders cite bank entry IDs** (our D1) — their Phase 2 explicitly allows synthesis beyond verbatim entries; we add citation for auditability.
- **Report** intervention rate + token/latency overhead (they only log it) — our D3, closes G9.
- Local open-weight models for both roles (their configs are OpenRouter frontier models; `litellm`/OpenAI-compatible endpoints make this a config change, and their SFT-model fallback parser suggests they already battle-tested small-model quirks).
