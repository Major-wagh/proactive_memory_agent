# Spec 001 — Memory Bank & Phase-1 Tool Surface

**Status:** v1 — accepted for M1, 2026-07-14. *Planning artifact: interface sketches below are normative contracts for M1; no code exists yet.*
**Implements:** FR-3, FR-4 of [spec 000](000_scope_and_decisions.md); defines the data shapes FR-8 will log.
**Closes gaps:** G3 (tag vocabulary), G4 (bank growth), G5 (access stats), G11 (entry update semantics).
**Grounding:** [part 04](../docs/context/part_04_memory_bank_and_tools.md) (paper §3.2–3.3) · [part 09](../docs/context/part_09_authors_reference_implementation.md) (reference implementation).

## 1. Purpose & scope

The memory bank is the **only mutable state** in the system and the four Phase-1 operations are its **only mutation surface**. This spec defines a pure data layer: no LLM calls, no prompts, no I/O beyond JSON (de)serialization. Everything here is testable without a model.

**Non-goals (owned by spec 002+):** phase prompts, trigger, Phase-2 decision, injection, orchestration/step logging, BM25 prefilter execution.

## 2. Data model (normative)

**MemoryBank**

| Field | Type | Constraint |
|---|---|---|
| `status` | str | ≤ `max_status_chars` (default **2000**, per authors); over-length input truncated with a visible `…[truncated]` marker |
| `knowledge` | ordered list of MemoryEntry | insertion order preserved |
| `procedural` | ordered list of MemoryEntry | insertion order preserved |

**MemoryEntry**

| Field | Type | Constraint |
|---|---|---|
| `id` | str | opaque, unique within a run, **never reused** (§3) |
| `content` | str | non-empty; ≤ `max_entry_chars` (default **1000**, configurable) with truncation marker; tag conventions §5 |
| `created_at` | str | ISO-8601 UTC |
| `created_step` | int \| null | **our addition** — action-step number at save time (D-001-3) |
| `access_count` | int | schema-compat with authors' JSON; **always 0 in v1** (never maintained) |
| `last_accessed` | str \| null | schema-compat; **always null in v1** |

**BankConfig:** `max_status_chars=2000`, `max_entry_chars=1000`. Surfaced through the run config in spec 002.

## 3. Identifier scheme (D-001-1)

Sequential, category-prefixed, monotonic: `K1, K2, …` for knowledge, `P1, P2, …` for procedural. Deletion never frees a number. When loading a foreign bank (e.g., authors'-format JSON with shortuuid ids), foreign ids are accepted opaquely and the counters seed past any colliding `K<n>`/`P<n>` pattern.

**Rationale:** deviation D1 (spec 000) requires Phase-2 reminders to cite entry ids. A 4B-class local memory model (see spec 000 OI-2) reproduces `K3` far more reliably than an 8-char `shortuuid` like `f4kPq2Zx`. Non-reuse guarantees a stale citation can never silently resolve to the wrong entry.

## 4. Operations — the only mutation surface (FR-3)

| Op | Args | Effect | Failure modes |
|---|---|---|---|
| `memory_update_status` | `content: str` | Replace status (truncating per §2). Empty string is legal (clears status). | — |
| `memory_save_knowledge` | `content: str` | Append new knowledge entry; returns new id. | empty/whitespace content → failed op, bank unchanged |
| `memory_save_procedural` | `content: str` | Append new procedural entry; returns new id. | empty/whitespace content → failed op, bank unchanged |
| `memory_delete` | `memory_id: str` | Remove entry from whichever category holds it. | unknown id → `success=False`, bank unchanged (no exception — authors' behavior) |

Each executed op yields a **MemoryOperation** record for spec 002's logging:
`{action, memory_id?, content?, success, truncated?, error?}`.

**Execution contract:**

- **EC-1** The orchestrator passes an **ordered list** of ops; they execute strictly in sequence.
- **EC-2** Per-op atomicity: a failed op is recorded and execution **continues** with the next op.
- **EC-3** An empty op list leaves the bank **serialization-identical**.
- **EC-4** No other mutation path exists (constructor + `load` + these four ops). The paper's "no free-form rewrite" property (FR-3) holds **by construction**, not by prompt discipline.
- **EC-5** There is **no in-place edit op** (G11): updating an entry = delete + save, which produces a **new** id. Ids are never resurrected.

## 5. Content conventions (G3)

Tags `[ENV] [PATH] [FACT]` (knowledge) and `[BUG] [PERF]` (procedural) are **prompt-level conventions** (spec 002 embeds them in the Phase-1 prompt, as the authors do). The bank never parses, validates, or enforces tags; untagged content is legal. An optional read-only helper `tag_of(entry)` may exist for stats.

## 6. Renders (consumed by spec 002)

- **`render_for_memory_agent(bank)`** — the authors' `<memory_bank>` XML: `<status>` + `<knowledge>`/`<procedural>` sections with one `[id] content` line per entry, `(empty)` placeholders. Deterministic given bank state.
- **`render_for_actor(bank)`** — knowledge + procedural only; **structurally status-free** (the render path must not have access to the status value — enforced by design and by AC-3). Rationale: part 09's war story — status text like "Task completed successfully" caused premature actor termination. Used only by ablation arms (full-bank exposure) and debugging, never by the main loop.

## 7. Growth policy (G4, D-001-5)

**v1: no eviction.** Growth is bounded in practice by `max_entry_chars` and proof-task length. The authors' approach beyond ~50 entries (BM25 top-20 prefilter of what the *memory agent's prompt* shows) is **deferred to v1.1** with an explicit trigger: adopt it the first time an M2/M3 run's bank exceeds ~50 entries. The bank exposes counts and char totals so the orchestrator can log growth per step (feeds G9 metering).

**Amendment (spec 002 §5, 2026-07-17):** on 4 GB VRAM with `num_ctx=4096` the **prompt token budget binds before the 50-entry count**. Spec 002's `ObservationBuilder` renders only a **recency-first subset** of entries that fits the bank-render slice (~900 tokens), upgrading to BM25-against-observation later. This is a **prompt-only view** — the bank remains the untruncated source of truth (the render helpers in §6 gain an optional `max_tokens`/selection arg; the four ops and serialization are unchanged).

## 8. Serialization

JSON, UTF-8:

```json
{
  "schema_version": "pma-bank/1",
  "status": "…",
  "knowledge":  [ { "id": "K1", "content": "…", "created_at": "…", "created_step": 1, "access_count": 0, "last_accessed": null } ],
  "procedural": [ … ]
}
```

- **Compatibility:** files shaped like the authors' `UniversalMemory.to_dict()` (no `schema_version`, shortuuid ids, no `created_step`) must load; missing fields get defaults, unknown fields are ignored.
- **Round-trip law:** `load(save(B)) ≡ B` (deep equality: ids, order, timestamps).
- One `memory.json` per run (final bank). Step-level op logs are spec 002's JSONL, not the bank's concern.

## 9. Metrics accessors (FR-8 slice)

Read-only accessors: entry counts per category, total content chars per category, serialized size. The bank itself never logs — the orchestrator (spec 002) reads these and writes the run log.

## 10. Acceptance criteria (become M1 unit tests verbatim)

| # | Criterion |
|---|---|
| AC-1 | Order matters: `[save_knowledge X, delete(id_X)]` → empty bank; `[delete(id_X), save_knowledge X]` → one entry + one failed op |
| AC-2 | Empty op list → identical serialization before/after |
| AC-3 | Status privacy: for arbitrary bank states (sentinel string in status), `render_for_actor` output never contains the sentinel |
| AC-4 | Status longer than `max_status_chars` is truncated with the marker |
| AC-5 | `memory_delete` of unknown id → `success=False`, bank unchanged |
| AC-6 | Ids unique and never reused: save → `K1`, delete `K1`, save → `K2` |
| AC-7 | JSON round-trip identity; an authors'-shape fixture (hand-built to `UniversalMemory.to_dict()` shape) loads and re-saves as `pma-bank/1` |
| AC-8 | Empty/whitespace content save → failed op, bank unchanged |
| AC-9 | Over-length entry content truncated at `max_entry_chars`; op `success=True`, `truncated=True` |
| AC-10 | Mutation-surface review: public API exposes no mutator beyond the four ops (+ constructor/load) |

## 11. Environment

Pure Python **3.12+ stdlib** (`dataclasses`, `json`, `datetime`) — **zero third-party dependencies** for the core (sequential ids remove the authors' `shortuuid` dependency). Developed Windows-first (owner's machine); CI on Linux later.

## 12. Decision log

| # | Decision | Rationale |
|---|---|---|
| D-001-1 | Sequential citable ids (`K1`/`P1`) instead of authors' 8-char shortuuid | Small-model citation reliability for deviation D1; paper only says "short identifier" |
| D-001-2 | `access_count`/`last_accessed` kept in schema, never maintained | Authors deliberately disabled tracking as misleading; keeps their JSON loadable without carrying dead behavior |
| D-001-3 | `created_step` added | Cheap provenance for debugging and future pivot-turn labeling (part 07 §7) |
| D-001-4 | Tags are convention, not schema | Matches authors; keeps the core validation-free |
| D-001-5 | No eviction in v1; BM25 prefilter deferred with explicit adoption trigger | Proof-eval tasks expected ≤ 50 entries; avoid speculative machinery |
| D-001-6 | `max_entry_chars=1000` guardrail (authors: uncapped) | Mechanically enforces the paper's "compact" entries; protects a 4 GB-VRAM context budget from runaway saves by a small memory model |

## 13. Handoffs to spec 002

- Char/token budget for the **memory-agent context** on 4 GB-class local models (task description + window formatting, per-step caps — the binding constraint on this hardware).
- Prompt-side bank cap behavior if the >50-entries trigger (§7) fires.
- `MemoryOperation` JSONL step-log schema (FR-8), including per-step token/latency fields (G9).
- `BankConfig` exposure through the run config.
