---
name: flow-trace-audit
description: Run an end-to-end reachability and flow-tracing audit on any codebase, based on the Debug Pilleur Master methodology by AVAA DEV. Use this skill whenever the user asks to audit wiring, reachability, dead code at boundaries, unconnected features, buttons that do nothing, endpoints never called, "is everything wired end to end", "câblage", "joignabilité", "trace le flux", "Debug Pilleur", or reports that features exist in code but do not work in the app. Also use it for ANY diagnostic or debugging task where the cause is not proven yet, when an AI agent ignores instructions, forgets, loops, calls tools that do not exist or claims work it did not do, and before any release when the user wants proof that every UI trigger reaches the backend and every backend capability is reachable. Works on ANY stack (Tauri, Electron, WebSocket, HTTP/REST, IPC, gRPC, tRPC, GraphQL, CLI, pipelines, AI agents) because the method discovers the project's real mechanisms before tracing.
---

# Flow-Trace Audit — Debug Pilleur Master

> **Tracing the flow IS the source of truth.** Not assumptions. Not memory. Not documentation. The real flow, proven line by line.
>
> "Un vrai debug trace le cheminement complet de bout en bout." Zero mock. Zero dead code. 100% of features connected. A diagnostic that does not trace the complete flow is an incomplete diagnostic — and therefore forbidden.

Methodology: **Debug Pilleur Master — Universal Debugging and Tracing Methodology** by **AVAA DEV** — L'agent IA gouverné. *NO CODE NO DONE.*

## REQUIRED FIRST STEP

Read the canonical methodology document bundled with this skill, in full:

**`references/DEBUG_PILLEUR_MASTER.md`** (framework-agnostic, French — canonical)

An English translation is available at `references/DEBUG_PILLEUR_MASTER_EN.md`. Read whichever matches the working language; the French version prevails in case of divergence.

It contains the complete per-flux procedures this skill applies: the 7 flux types with their step-by-step verification chains, checklists and schemas; the reference grep commands; the table of common failure patterns; the Règle #30 five-pass output-reading protocol; the forbidden behaviors; and how to use the method for development tasks as well as diagnostics. Do not proceed on a partial reading. The rest of this file tells you how to orchestrate that methodology into a reachability audit.

## Universal rule: TRACE BEFORE TOUCHING

Whatever the task, identify and verify every link of the chain BEFORE modifying anything. If a single link is missing, broken, or unverified, the diagnostic is incomplete and any modification is forbidden. This audit modifies zero project files: it produces a diagnostic report; fixes come after, as a separate task, ordered by severity.

## The 7 flux types (full procedures in the reference document)

Every software system decomposes into combinations of these seven flux. Identify which flux are concerned BEFORE tracing.

| Flux | Chain | Steps |
|------|-------|-------|
| A | User input → Backend (UI/CLI/API) | 6: entry point, transport, handler, business module, method signature, response handled |
| B | Backend → Backend | 4: caller, interface/signature, instantiation, return value |
| C | Backend → UI | 5: emitter, data format, transport channel, UI reception, rendering/binding |
| D | UI internal (components, state, navigation) | 3: data source, data chain, reactivity |
| E | Pipeline / Workflow | 5: trigger, ordered steps, inter-step interfaces (output N = input N+1), state/status, final output with proof |
| F | Configuration / Infra | 4: config file, consumers, key/type/value coherence, impact |
| G | Agent cognition (LLM, orchestrator, organ, multi-agent) | 7: perception, attention (real assembled context), recall, reasoning (traced decision), action, feedback (observation reinjected), consolidation |

## Audit workflow

### Phase 0 — Discovery
Never assume the stack. Read the project manifests, then locate the real communication mechanisms used in THIS project (the reference document's per-flux "quoi chercher" tables and reference commands are starting heuristics, not limits). Output: numbered list of mechanisms with one file:line example each and occurrence counts. Any mechanism you cannot map is flagged `UNMAPPED-MECHANISM`, never ignored. Also list every pipeline (flux E), every behavior-gating configuration (flux F) and every AI agent loop (flux G) found.

### Scale protocol (applies to all phases)
Inventories are persisted, never memorized: write them to a scratch directory OUTSIDE the project tree (e.g. `/tmp/flow-trace-<project>/`), one file per inventory, appended as items are discovered. On codebases too large for a single pass, segment by feature or top-level directory: run Phases 1–4 per segment with a per-segment integrity check, then merge into ONE global matrix whose arithmetic must balance across all segments — cross-segment calls are matched at merge time, never dropped. The zero-modification rule concerns the project's files only: audit scratch files outside the tree are allowed; the report is the sole deliverable.

### Phases 1–3 — Three exhaustive inventories
1. **Inventory A — backend entry points** (flux A steps 2–5, flux B step 2): every registered handler/route/command, with identifier, definition file:line, registration file:line, payload shape, observable effect. Defined-but-unregistered items get status `DEFINED-NOT-REGISTERED`.
2. **Inventory B — calls emitted by the frontend/CLI** (flux A step 1): every transport call site, identifier resolved to its literal value by reading its definition — never guessed. Runtime-computed identifiers get status `DYNAMIC-IDENTIFIER` with every possible value traced.
3. **Inventory C — UI triggers and their chains** (flux A step 1 + flux D): every user-facing trigger, followed function by function until it reaches a transport call, completes a legitimate frontend-only effect (`FRONTEND-ONLY`), or dead-ends.

State exact counts for each inventory. If you announce 47 entry points, the table has 47 rows.

### Phase 4 — Reachability matrix and anomaly classification
Cross-reference the three inventories. Every gap is classified into exactly one status:

| Code | Anomaly | Rule | Severity |
|------|---------|------|----------|
| `BROKEN-CALL` | Call targets no registered entry point | In B, absent from A | CRITICAL |
| `INERT-TRIGGER` | Trigger chain dead-ends before any effect | In C, dead-end | HIGH |
| `SWALLOWED-ERROR` | Matched pair whose error branch is empty or swallowed (`except: pass`, empty `.catch`) | Return-path check on matched pairs | HIGH |
| `DEAD-ENDPOINT` | Entry point never called | In A, absent from B | MEDIUM |
| `ORPHAN-RESPONSE` | Backend emits data no frontend handler receives (flux C steps 4–5) | Emitted identifiers absent from listeners | MEDIUM |
| `DEFINED-NOT-REGISTERED` | Handler/command defined but never registered on its transport | In A, registration absent (Phase 1 flag) | MEDIUM |
| `PIPELINE-BREAK` | Pipeline step whose output ≠ next step's input, or status never updated | Flux E checklist fails | HIGH |
| `CONFIG-DRIFT` | Config key consumed but absent, typo'd, type-mismatched, or duplicated with conflicting values | Flux F checklist fails | MEDIUM |
| `COGNITION-BREAK` | Agent cycle broken: truncated context, failed recall, decision short-circuited by a hardwired rule/keyword/template, hallucinated tool, ignored observation, no stop condition, or missing consolidation | Flux G checklist fails | HIGH |

Flux E, F and G: when Phase 0 discovers pipelines, behavior-gating configuration or agent loops, trace them with their own checklists from the reference document; their failures enter the same anomaly list as `PIPELINE-BREAK` / `CONFIG-DRIFT` / `COGNITION-BREAK`. Each `COGNITION-BREAK` names the failing flux G step (1–7).

Non-anomaly statuses land as follows: `FRONTEND-ONLY` items are matrix rows with that status (legitimate, counted); each `DYNAMIC-IDENTIFIER` call site produces one matrix row per resolved value; `UNMAPPED-MECHANISM` findings are counted in report section 0 AND in the integrity arithmetic.

For every matched pair, verify the full return path per flux C: format, channel, reception handler, rendering/binding — success AND error branches.

### Phase 5 — Reading every result: Règle #30 (non-negotiable)
Every command output produced during this audit is read with the five-pass protocol from the reference document: (1) raw count — total lines, negatives, positives; (2) line-by-line word-by-word reading, flagging corruption, unexpected zeros/NULLs, duplicates, incoherences immediately; (3) cross-referencing — totals balance, expected-but-absent data identified; (4) agent-lie detection — "FAIT" without proof, embellished counts, `| head` shown as global proof, redefined success criteria; (5) implications and exact corrective actions. Then score: `SCORE: X/Y lignes propres` and verdict `PROPRE | SUSPECT | CORROMPU`. One critical line forbids a PROPRE verdict.

### Phase 6 — Report
```
# FLOW-TRACE AUDIT REPORT — Debug Pilleur Master
Project: <name> | Date: <date> | flow-trace-audit v1.4.0 (AVAA DEV)

## 0. Flux types concerned + mechanisms discovered (with file:line evidence and counts)
## 1. Counts — A: N | B: M | C: K | matched chains: X | anomalies: Y | coverage: X/K (P%)
## 2. Reachability matrix (one row per trigger: trigger → call → entry point → effect → response handled → status)
## 3. Anomalies ordered CRITICAL → HIGH → MEDIUM, each with [CODE], identifier, file:line, exact quoted line, failure cause, wiring direction (no code changed)
## 4. Checklists — the per-flux checklist of every flux traced, every box proven
## 5. Integrity check — every item of A, B, C appears exactly once in matrix or anomalies; state the arithmetic
```
The audit is complete ONLY when section 5 balances. If it does not, the audit itself has a bug: find the dropped items before delivering. Before delivering, apply flux G to your own audit work (reference document, "Pour conclure toute tache"): every claim in the report matches an observation you actually made.

## Development tasks

The same method governs building, not only diagnosing. When the user asks to build or modify a feature, follow the reference document's "Pour chaque tache de developpement" sequence: identify the flux touched, trace the existing chain, write the target chain, implement link by link, prove every checklist box, and declare done only when the target chain is traced end to end. In that mode project files are modified as the task requires; the zero-modification rule above applies to audits only.

## Forbidden behaviors (enforced; full list in reference document)

Reading laziness (reading 30 of 500 lines and concluding; "the rest follows the same pattern"), diagnosis by assumption ("this method should exist" — grep first; "the agent understood the instruction" — read the context actually sent), reactive debugging (running first, diagnosing after the crash), false diagnostics ("everything works" without execution proof; partial results shown as global proof), and excessive conservatism (asking permission at every micro-step when the instruction is explicit — if in blocking doubt, ONE single stop).

---

Final critical constraints: read `references/DEBUG_PILLEUR_MASTER.md` in full before starting; identify flux types before tracing; discover mechanisms before inventorying; persist inventories outside the project tree; resolve every identifier to its literal value; apply Règle #30 five passes to every output; cite file:line with the exact quoted line for every claim; classify every gap into the nine anomaly statuses; modify zero project files during an audit; apply flux G to your own work before concluding; deliver the report only when the integrity arithmetic balances globally.
