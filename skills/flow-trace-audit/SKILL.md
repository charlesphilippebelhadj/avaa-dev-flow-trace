---
name: flow-trace-audit
description: Run an end-to-end reachability and flow-tracing audit on any codebase, based on the Debug Pilleur Master methodology by AVAA DEV. Use this skill whenever the user asks to audit wiring, reachability, dead code at boundaries, unconnected features, buttons that do nothing, endpoints never called, "is everything wired end to end", "câblage", "joignabilité", "trace le flux", "Debug Pilleur", or reports that features exist in code but do not work in the app. Also use it for ANY diagnostic or debugging task where the cause is not proven yet, and before any release when the user wants proof that every UI trigger reaches the backend and every backend capability is reachable. Works on ANY stack (Tauri, Electron, WebSocket, HTTP/REST, IPC, gRPC, tRPC, GraphQL, CLI, pipelines) because the method discovers the project's real mechanisms before tracing.
---

# Flow-Trace Audit — Debug Pilleur Master

> **Tracing the flow IS the source of truth.** Not assumptions. Not memory. Not documentation. The real flow, proven line by line.
>
> "Un vrai debug trace le cheminement complet de bout en bout." Zero mock. Zero dead code. 100% of features connected. A diagnostic that does not trace the complete flow is an incomplete diagnostic — and therefore forbidden.

Methodology: **Debug Pilleur Master** by **AVAA DEV** — L'agent IA gouverné. *NO CODE NO DONE.*

## REQUIRED FIRST STEP

Read the canonical methodology document bundled with this skill, in full:

**`references/DEBUG_PILLEUR_MASTER.md`** (760 lines, framework-agnostic, French — canonical)

An English translation is available at `references/DEBUG_PILLEUR_MASTER_EN.md`. Read whichever matches the working language; the French version prevails in case of divergence.

It contains the complete per-flux procedures this skill applies: the 6 flux types with their step-by-step verification chains, checklists and schemas; the reference grep commands; the table of 18 common failure patterns; the Règle #30 five-pass output-reading protocol; and the forbidden behaviors. Do not proceed on a partial reading. The rest of this file tells you how to orchestrate that methodology into a reachability audit.

## Universal rule: TRACE BEFORE TOUCHING

Whatever the task, identify and verify every link of the chain BEFORE modifying anything. If a single link is missing, broken, or unverified, the diagnostic is incomplete and any modification is forbidden. This audit modifies zero files: it produces a diagnostic report; fixes come after, as a separate task, ordered by severity.

## The 6 flux types (full procedures in the reference document)

Every software system decomposes into combinations of these six flux. Identify which flux are concerned BEFORE tracing.

| Flux | Chain | Steps |
|------|-------|-------|
| A | User input → Backend (UI/CLI/API) | 6: entry point, transport, handler, business module, method signature, response handled |
| B | Backend → Backend | 4: caller, interface/signature, instantiation, return value |
| C | Backend → UI | 5: emitter, data format, transport channel, UI reception, rendering/binding |
| D | UI internal (components, state, navigation) | 3: data source, data chain, reactivity |
| E | Pipeline / Workflow | 5: trigger, ordered steps, inter-step interfaces (output N = input N+1), state/status, final output with proof |
| F | Configuration / Infra | 4: config file, consumers, key/type/value coherence, impact |

## Audit workflow

### Phase 0 — Discovery
Never assume the stack. Read the project manifests, then locate the real communication mechanisms used in THIS project (the reference document's per-flux "quoi chercher" tables and reference commands are starting heuristics, not limits). Output: numbered list of mechanisms with one file:line example each and occurrence counts. Any mechanism you cannot map is flagged `UNMAPPED-MECHANISM`, never ignored.

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

For every matched pair, verify the full return path per flux C: format, channel, reception handler, rendering/binding — success AND error branches.

### Phase 5 — Reading every result: Règle #30 (non-negotiable)
Every command output produced during this audit is read with the five-pass protocol from the reference document: (1) raw count — total lines, negatives, positives; (2) line-by-line word-by-word reading, flagging corruption, unexpected zeros/NULLs, duplicates, incoherences immediately; (3) cross-referencing — totals balance, expected-but-absent data identified; (4) agent-lie detection — "FAIT" without proof, embellished counts, `| head` shown as global proof, redefined success criteria; (5) implications and exact corrective actions. Then score: `SCORE: X/Y lignes propres` and verdict `PROPRE | SUSPECT | CORROMPU`. One critical line forbids a PROPRE verdict.

### Phase 6 — Report
```
# FLOW-TRACE AUDIT REPORT — Debug Pilleur Master
Project: <name> | Date: <date> | flow-trace-audit v1.2.0 (AVAA DEV)

## 0. Flux types concerned + mechanisms discovered (with file:line evidence and counts)
## 1. Counts — A: N | B: M | C: K | matched chains: X | anomalies: Y | coverage: X/K (P%)
## 2. Reachability matrix (one row per trigger: trigger → call → entry point → effect → response handled → status)
## 3. Anomalies ordered CRITICAL → HIGH → MEDIUM, each with [CODE], identifier, file:line, exact quoted line, failure cause, wiring direction (no code changed)
## 4. Checklists — the per-flux checklist of every flux traced, every box proven
## 5. Integrity check — every item of A, B, C appears exactly once in matrix or anomalies; state the arithmetic
```
The audit is complete ONLY when section 5 balances. If it does not, the audit itself has a bug: find the dropped items before delivering.

## Forbidden behaviors (enforced; full list in reference document)

Reading laziness (reading 30 of 500 lines and concluding; "the rest follows the same pattern"), diagnosis by assumption ("this method should exist" — grep first), reactive debugging (running first, diagnosing after the crash), false diagnostics ("everything works" without execution proof; partial results shown as global proof), and excessive conservatism (asking permission at every micro-step when the instruction is explicit — if in blocking doubt, ONE single stop).

---

Final critical constraints: read `references/DEBUG_PILLEUR_MASTER.md` in full before starting; identify flux types before tracing; discover mechanisms before inventorying; resolve every identifier to its literal value; apply Règle #30 five passes to every output; cite file:line with the exact quoted line for every claim; classify every gap into the five statuses; modify zero files; deliver the report only when the integrity arithmetic balances.
