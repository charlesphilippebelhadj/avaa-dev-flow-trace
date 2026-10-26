# Debug Pilleur Master — Universal Debugging and Tracing Methodology

> Reference document. To be integrated into the CLAUDE.md of every project.
> A working method as much as a debugging one: understand what is happening, decide what to do and how to do it.
> This protocol is NON-NEGOTIABLE for EVERY diagnostic, in EVERY project.
> Every bug, every modification, every wiring task begins with a complete flow trace.
> Tracing the flow IS the source of truth. Not assumptions. Not memory. The real flow.
>
> *English translation of the canonical French document `DEBUG_PILLEUR_MASTER.md` (universal version, 2026-05-24, flux G added 2026-10-26). In case of divergence, the French version prevails.*

## Table of contents

1. Fundamental Principle
2. The 7 flux types
3. FLUX A — User input → Backend (6 steps)
4. FLUX B — Backend → Backend (4 steps)
5. FLUX C — Backend → UI (5 steps)
6. FLUX D — UI internal (3 steps)
7. FLUX E — Pipeline / Workflow (5 steps)
8. FLUX F — Configuration / Infra (4 steps)
9. FLUX G — Agent cognition (7 steps)
10. Common problems (all flux)
11. Reference commands
12. Output-reading protocol — Rule #30
13. Forbidden behaviors
14. How to use this document

This document is read in full, start to finish, before any audit or any task. This table exists solely for navigation while applying the procedures.

---

## Fundamental Principle

**"A real debug traces the complete path end to end."**

Zero mocks. Zero dead code. 100% of features connected. A diagnostic that does not trace the complete flow = an incomplete diagnostic = FORBIDDEN.

### Universal rule: TRACE BEFORE TOUCHING

Whatever the type of modification, IDENTIFY and VERIFY every link of the chain BEFORE modifying anything. If a single link is missing, broken, or unverified, the diagnostic is incomplete and any modification is FORBIDDEN.

---

## The 7 flux types

Every software system decomposes into combinations of these 7 flux. Identify the relevant flux BEFORE debugging.

| Flux | Description | When to use it |
|------|-------------|----------------|
| A | User input → Backend | Action triggered from the UI or CLI toward the backend |
| B | Backend → Backend | Module/service calling another module/service |
| C | Backend → UI | Data produced by the backend and displayed in the interface |
| D | UI internal | Components, state management, frontend navigation |
| E | Pipeline / Workflow | Chain of sequential processing steps (CI/CD, agents, batch) |
| F | Configuration / Infra | Configs, schemas, environment variables, permissions, hooks |
| G | Agent cognition | AI agent (LLM, multi-agent, cognitive architecture): what it perceives, retains, decides, does and learns |

---

## FLUX A — User input → Backend (6 steps)

For every feature triggered by the user (click, command, request).

### 1. Entry point identified

Where does the user trigger the action?

```bash
# Web UI — network calls
grep -rn "fetch\|axios\|sendMessage\|emit\|submit\|onClick\|dispatch" src/ app/ frontend/

# CLI — commands
grep -rn "argparse\|click\|typer\|sys.argv\|command" cli/ commands/ tools/

# API — endpoints
grep -rn "route\|endpoint\|@app\.\|@router\." api/ routes/ controllers/ views/
```

### 2. Transport verified

Through which mechanism does the request reach the backend?

| Transport | What to look for |
|-----------|------------------|
| REST API | `@app.get`, `@router.post`, `route()` |
| WebSocket | `register_handler`, `on_message`, `subscribe` |
| Direct CLI | `def main`, `if __name__`, commands |
| IPC / Event bus | `emit`, `publish`, `subscribe`, `on()` |
| Message queue | `consume`, `produce`, `channel` |

```bash
grep -rn "register_handler\|@app\.\|@router\.\|route(" server/ api/ core/
```

### 3. Handler/controller located

Does the code that receives and processes the request exist?

```bash
grep -rn "def handler_name\|def action_name\|async def" handlers/ controllers/ core/
```

### 4. Backend module called

Which business module does the handler call?

```bash
# Follow the imports and calls inside the handler
grep -n "from\|import\|self\._get\|self\." handlers/handler_file.py
```

### 5. Backend method verified

Does the called method exist AND does its signature match the call?

```bash
grep -n "def method_name(" module_file.py
# Compare params with the real call in the handler
```

### 6. Response handled by the UI

Does the UI/CLI correctly handle the returned response?

```bash
grep -rn "data.action\|response\.\|then(\|catch(\|onMessage\|handleResponse" src/ frontend/
```

### Flux A checklist

- [ ] User entry point identified (UI/CLI/API)
- [ ] Transport verified (REST/WS/CLI/IPC)
- [ ] Handler/controller exists and receives the request
- [ ] Business module called correctly
- [ ] Backend method exists with a conforming signature
- [ ] Response handled on the UI/CLI side

### Diagram

```
+-------------------+
|      User         |  1. Entry point
|  click/cmd/request|
+---------+---------+
          |
          v
+---------+---------+
|    Transport      |  2. REST/WS/CLI/IPC
|   route/handler   |
+---------+---------+
          |
          v
+---------+---------+
|    Handler        |  3. Controller
|  receives + parses|
+---------+---------+
          |
          v
+---------+---------+
|  Business module  |  4-5. Backend logic
|   method()        |
+---------+---------+
          |
          v
+---------+---------+
|    Response       |  6. Return to UI/CLI
|  JSON/stdout/evt  |
+-------------------+
```

---

## FLUX B — Backend → Backend (4 steps)

For every modification involving modules/services calling each other.

### 1. Caller identified

Who calls the function/method/service?

```bash
grep -rn "module_name\.\|from.*module_name import" src/ core/ lib/
```

### 2. Interface verified

Does the signature match what the caller sends?

```bash
# Read the real signature
grep -n "def method_name(" module_name.py

# Compare with the real call
grep -n "module_name.method_name(" caller.py
```

Points to verify:
- Identical number of parameters
- Compatible types
- sync vs async (`await` present if async)
- Optional parameters with coherent defaults

### 3. Instantiation verified

Is the module instantiated/accessible at the moment of the call?

```bash
# Singleton? Factory? Registry? DI container? Direct import?
grep -rn "ModuleClass(\|get_instance\|registry\|inject\|@singleton" src/ core/
```

### 4. Return value verified

Is what the method returns what the caller expects?

- Return type (dict, list, str, object, possible None?)
- Structure (which keys? which fields?)
- Error cases (exception? None? empty dict? error code?)

### Flux B checklist

- [ ] All callers identified (exhaustive grep)
- [ ] Signature conforming (params, types, sync/async)
- [ ] Module instantiated and accessible at runtime
- [ ] Return value conforming to what the caller consumes

### Diagram

```
+-------------------+
|    Module A       |  1. Caller
|  caller.method    |
+---------+---------+
          |
          | import / call
          v
+---------+---------+
|    Interface      |  2. Signature verified
|  def method(...)  |
+---------+---------+
          |
          | instantiation
          v
+---------+---------+
|  Instantiation    |  3. Singleton/DI/registry
|   accessible ?    |
+---------+---------+
          |
          | return
          v
+---------+---------+
|    Module A       |  4. Consumes the return
|  uses result      |
+-------------------+
```

---

## FLUX C — Backend → UI (5 steps)

For every modification involving a backend module sending data to the interface.

### 1. Emitter identified

Which backend module produces the data?

```bash
grep -rn "send\|emit\|response\|broadcast\|publish\|return.*json\|jsonify\|render" src/ core/ api/ controllers/
```

### 2. Data format verified

Does the sent structure match what the UI expects?

- List EVERY key/field sent
- Compare with what the frontend actually reads
- Types: string vs number vs boolean vs object vs array

### 3. Transport channel verified

Through which mechanism does the data travel?

| Transport | What to look for |
|-----------|------------------|
| REST response | `return jsonify`, `Response()`, status codes |
| WebSocket push | `send_json`, `emit`, `broadcast`, action name |
| SSE | `event-stream`, `yield` |
| File/stdout | `print`, `write`, `save` |

```bash
grep -rn "send_response\|send_json\|jsonify\|emit\|broadcast\|action.*=" src/ core/
```

### 4. UI reception verified

Does the frontend have a handler for this data?

```bash
grep -rn "fetch\|axios\|onMessage\|subscribe\|data\.action\|addEventListener\|useQuery" src/ frontend/ app/
```

### 5. Rendering verified

Does the component correctly display the received data?

- Right field (the key name on the UI side = the name on the backend side)
- Right format (string expected, not an object)
- Reactive binding (not a disconnected static copy)
- Error and loading states handled

### Flux C checklist

- [ ] Emitting module identified
- [ ] Sent data structure documented (every key listed)
- [ ] Transport channel identified and verified
- [ ] UI reception handler exists
- [ ] Correct rendering/binding in the component

### Diagram

```
+-------------------+
|  Backend module   |  1. Emitter
|  send/emit/return |
+---------+---------+
          |
          | JSON/dict/object
          v
+---------+---------+
|   Data format     |  2. Structure verified
|  keys, types      |
+---------+---------+
          |
          | REST/WS/SSE/IPC
          v
+---------+---------+
| Transport channel |  3. Mechanism
|   verified        |
+---------+---------+
          |
          v
+---------+---------+
|   UI reception    |  4. Frontend handler
|  fetch/onMessage  |
+---------+---------+
          |
          v
+---------+---------+
|   Rendering       |  5. Component
|  binding, display |
+-------------------+
```

---

## FLUX D — UI internal (3 steps)

For every purely frontend modification (components, state management, navigation).

### 1. Data source identified

Where does the displayed data come from?

```bash
grep -rn "useStore\|useState\|defineProps\|computed\|ref(\|reactive(\|createStore\|useSelector\|mapState" src/ frontend/ app/
```

- Global store (Pinia, Redux, Vuex, Zustand)?
- Props from the parent component?
- Local state (ref, useState)?
- Direct network call?
- Context/injection (provide/inject, React context)?

### 2. Data flow traced

From the source to the component that displays it: every link verified.

```bash
grep -rn "import.*store\|import.*component\|provide\|inject\|useContext\|connect(" src/ frontend/
```

### 3. Reactivity verified

Are the bindings reactive? Do updates propagate?

- No raw copy disconnected from the state
- The component re-renders when the data changes
- Watchers/subscriptions in place if needed
- No direct state mutation (if immutable pattern)

### Flux D checklist

- [ ] Data source identified (store, props, local, network)
- [ ] Data chain traced end to end
- [ ] Reactivity confirmed

### Diagram

```
+-------------------+
|   Data source     |  1. Store/Props/Local/API
|   identified      |
+---------+---------+
          |
          | import/props/inject
          v
+---------+---------+
|   Data flow       |  2. Chain traced
|  parent → child   |
+---------+---------+
          |
          | ref/computed/watch
          v
+---------+---------+
|   Reactivity      |  3. Binding verified
|  correct render   |
+-------------------+
```

---

## FLUX E — Pipeline / Workflow (5 steps)

For every chain of sequential processing (CI/CD, agentic pipeline, batch processing, ETL).

### 1. Trigger identified

What starts the pipeline?

```bash
grep -rn "trigger\|schedule\|cron\|main\|entry_point\|start\|run_pipeline\|on_event" pipeline/ workflow/ ci/ .github/
```

### 2. Steps listed and ordered

Which steps, in which order, with which dependencies?

```bash
grep -rn "step\|stage\|phase\|task\|job\|substep" pipeline/ workflow/ config/
```

### 3. Inter-step interface verified

Does each step receive the output of the previous one? Compatible format?

- Output of step N = Input of step N+1
- Compatible types/schemas
- Error handling: what happens if a step fails?

### 4. State / status verified

Is the pipeline state correctly updated at each step?

```bash
grep -rn "status\|state\|FAIT\|EN_COURS\|FAILED\|DONE\|PENDING" pipeline/ state/ config/
```

### 5. Final output verified

Is the final result what is expected? Artifacts produced? Proof present?

### Flux E checklist

- [ ] Trigger identified
- [ ] All steps listed in order
- [ ] Inter-step interfaces compatible (output N = input N+1)
- [ ] State/status updated correctly
- [ ] Final output conforming + proof present

### Diagram

```
+-------------------+
|    Trigger        |  1. Trigger/cron/event
|  start pipeline   |
+---------+---------+
          |
          v
+---------+---------+
|    Step 1         |  2. Listed + ordered
|  input → output   |
+---------+---------+
          |
          | output = next input
          v
+---------+---------+
|    Step N         |  3. Interface verified
|  input → output   |
+---------+---------+
          |
          v
+---------+---------+
|  Pipeline state   |  4. Status updated
|  DONE/FAILED      |
+---------+---------+
          |
          v
+---------+---------+
|  Final output     |  5. Artifacts + proof
|  real deliverable |
+-------------------+
```

---

## FLUX F — Configuration / Infra (4 steps)

For every modification touching configs, schemas, permissions, environment variables, hooks.

### 1. Config file identified

Which configuration file is concerned?

```bash
grep -rn "config\|settings\|env\|schema\|permission" config/ .env .config/ settings/
```

### 2. Consumers identified

Which modules/services read this config?

```bash
grep -rn "config_key\|getenv\|load_config\|read_settings" src/ core/ lib/
```

### 3. Coherence verified

Is the value in the config coherent with what the consumers expect?

- Existing key (no typo)
- Correct type (string vs int vs bool)
- Value within the expected bounds
- No duplicate config with different values

### 4. Impact traced

Which behaviors change if this config changes?

```bash
grep -rn "if.*config_key\|when.*setting\|gate\|hook\|permission" src/ core/ hooks/
```

### Flux F checklist

- [ ] Config file identified
- [ ] All consumers identified (exhaustive grep)
- [ ] Key/type/value coherence verified
- [ ] Impact of the change traced

### Diagram

```
+-------------------+
|   Config file     |  1. .env/yaml/json/toml
|   key = value     |
+---------+---------+
          |
          | read by
          v
+---------+---------+
|   Consumers       |  2. Modules that read
|   load/getenv     |
+---------+---------+
          |
          | applied
          v
+---------+---------+
|   Coherence       |  3. Type/value/bounds
|   no conflict     |
+---------+---------+
          |
          v
+---------+---------+
|   Impact          |  4. Behaviors
|   gates/hooks/if  |
+-------------------+
```

---

## FLUX G — Agent cognition (7 steps)

For every behavior of an AI agent (LLM, orchestrator, organ, sub-agent, multi-agent system): an agent is a flux like any other. What it "understands" is never assumed — it is what actually went through each step of the cognitive cycle. Flux G combines with the others: an agent action is often a flux A or B, its memory a flux F or E.

### 1. Perception — stimulus identified

What did the agent actually receive?

```bash
grep -rn "on_message\|handle_input\|user_message\|receive\|on_event\|stimulus" agents/ core/ src/
```

- User message, event, result from another agent, scheduled trigger
- Exact content received (not the content you believe was sent)
- Source and format identified

### 2. Attention — assembled context verified

What actually enters the context window at reasoning time?

```bash
grep -rn "system_prompt\|messages\|build_context\|context_window\|max_tokens\|truncate" agents/ core/ src/
```

- REAL content sent to the model (logged payload, not reconstructed)
- Measured size vs limit: truncation, pruning, automatic summarization?
- Order of elements (instructions, history, documents, declared tools)
- No expected instruction or data missing from the context

### 3. Recall — retrieved memory verified

What did the agent retrieve from its memory, and why?

```bash
grep -rn "retrieve\|recall\|search_memory\|embedding\|vector\|top_k\|similarity" agents/ memory/ core/ src/
```

- Actual recall query (text, filters, threshold, number of results)
- Retrieved items read in full — relevant, up to date, non-contradictory
- Expected but NOT retrieved items identified
- The recall is injected into the context (step 2), not merely computed

### 4. Reasoning — decision traced

On what basis did the agent choose what to do?

```bash
grep -rn "plan\|decide\|route\|select_tool\|tool_choice\|intent" agents/ core/ src/
```

- Decision derived from reasoning over the context (steps 1 to 3), not from a hardwired rule, a keyword or a template that short-circuits cognition
- Observable reasoning or plan (trace, log) read in full
- Choice coherent with the instructions and the data present
- Discarded alternatives identifiable when the trace allows it

### 5. Action — execution verified

Did the decided action actually take place, as decided?

```bash
grep -rn "tool_calls\|tool_use\|function_call\|execute_tool\|invoke\|dispatch" agents/ tools/ core/ src/
```

- Called tool/organ exists in the declared registry (no hallucinated tool)
- Arguments conform to the declared schema (names, types, required fields)
- Real execution (not simulated, not mocked) — proof in the logs
- If the action crosses a transport: trace the corresponding flux A or B

### 6. Feedback — observation integrated

Does the action's result come back into the loop, and does the agent take it into account?

```bash
grep -rn "tool_result\|observation\|on_result\|feedback\|max_iterations\|stop_condition" agents/ core/ src/
```

- Result reinjected into the context of the next iteration
- Errors propagated to the agent, not swallowed (`except: pass`, empty result)
- Stop condition present (goal reached, budget, number of iterations)
- Final output grounded in the observations: every claim the agent makes ("DONE", "fixed", "tested") matches a real result

### 7. Consolidation — memory updated

Is what the agent learned persisted and retrievable?

```bash
grep -rn "save_memory\|persist\|consolidate\|write_memory\|upsert\|store" agents/ memory/ core/ src/
```

- Write actually performed (proof: re-reading the storage)
- Written content = what was observed, not an assumption by the agent
- Retrievable by the recall of step 3 on the next cycle (closed loop)
- No contradictory duplicate of an existing memory

### Flux G checklist

- [ ] Stimulus actually received identified (exact content, source)
- [ ] Real assembled context verified (content, size, truncation)
- [ ] Memory recall verified (query, results, absences)
- [ ] Decision traced back to the reasoning, with no hardwired rule short-circuiting it
- [ ] Action executed in conformity with the registry and schema, with proof
- [ ] Observation reinjected, errors propagated, stop condition present
- [ ] Memory consolidated and retrievable on the next cycle

### Diagram

```
+-------------------+
|   Perception      |  1. Stimulus
|   actual input    |
+---------+---------+
          |
          v
+---------+---------+
|   Attention       |  2. Context
|   actual context  |
+---------+---------+
          |
          v
+---------+---------+
|   Recall          |  3. Recall
|   memory          |<--------------------------+
+---------+---------+                           |
          |                                     |
          v                                     |
+---------+---------+                           |
|   Reasoning       |  4. Decision              |
|   traced decision |<----------------+         |
+---------+---------+                 |         |
          |                           |         |
          v                           |         |
+---------+---------+                 |         |
|   Action          |  5. Execution   |         |
|   tool/organ      |                 |         |
+---------+---------+                 |         |
          |                           |         |
          v                           |         |
+---------+---------+                 |         |
|   Feedback        |  6. Observation |         |
|   actual result   |-----------------+         |
+---------+---------+                           |
          |                                     |
          v                                     |
+---------+---------+                           |
|   Consolidation   |  7. Memory                |
|   learning        |---------------------------+
+-------------------+
                        inner loop: iteration
                        outer loop: next cycle
```

---

## Common problems (all flux)

| Problem | Symptom | Flux | Solution |
|---------|---------|------|----------|
| Missing handler | Action silently ignored | A | Add the handler/route |
| Nonexistent method | AttributeError / 404 | A, B | Implement the method |
| Forgotten mock | Static data returned | A, B | Remove the mock, call the real module |
| Missing UI handler | UI does not react | A, C | Add the response handler |
| Uninitialized singleton | Module returns None/undefined | B | Verify instantiation |
| Incompatible signature | TypeError at call time | B | Compare def vs the real call |
| Circular import | ImportError at startup | B | Lazy import or injection |
| Missing JSON key | undefined/KeyError on the UI side | C | Compare sent keys vs read keys |
| Non-reactive data | UI does not update | C, D | Verify reactive bindings |
| Uninitialized store | undefined in the component | D | Verify store initialization |
| Broken pipeline step | Chain stops silently | E | Verify output N = input N+1 |
| Status never updated | Pipeline stuck EN_COURS (in progress) | E | Add status update |
| Config typo | Default value taken wrongly | F | Verify the exact key name |
| Duplicate config | Unpredictable behavior | F | Deduplicate, single source |
| Non-awaited async | Promise instead of value | B | Add await, verify async/def |
| Silent transport | Data sent, never received | C | Verify channel + reception handler |
| Incompatible format | TypeError on the consumer side | B, C | Compare sent type vs expected type |
| Missing proof | Status DONE without evidence | E | Collect proof before marking |
| Truncated context | Agent ignores a provided instruction or data | G | Measure the real context sent, check truncation and order |
| Failed recall | Agent asks again a question already resolved | G | Verify recall query, threshold, injection into the context |
| Short-circuited decision | Same behavior whatever the context | G | Trace the decision: hardwired rule, keyword or template instead of reasoning |
| Hallucinated tool | Call to a nonexistent tool or parameter | G | Compare the call with the declared registry and schema |
| Ignored observation | Answer contradicts the tool result | G | Verify the result is reinjected into the next iteration |
| Endless loop | Agent repeats the same action | G | Verify stop condition and iteration budget |
| Missing consolidation | Agent forgets from one session to the next | G | Verify memory write, then re-read on the next cycle |

---

## Reference commands

```bash
# === FLUX A — User inputs ===

# UI/CLI entry points
grep -rn "onClick\|submit\|sendMessage\|fetch\|axios\|command" src/ frontend/ cli/

# Routes/endpoints
grep -rn "@app\.\|@router\.\|route(\|register_handler" api/ server/ core/

# === FLUX B — Internal backend ===

# Who calls a module
grep -rn "from.*module_name import\|module_name\." src/ core/ lib/

# Where a module is instantiated
grep -rn "ModuleClass(\|get_instance\|@singleton" src/ core/

# Sync/async
grep -n "async def\|def " module_name.py

# === FLUX C — Backend → UI ===

# Sends toward the frontend
grep -rn "send_response\|send_json\|jsonify\|emit\|broadcast\|return.*json" src/ core/ api/

# Frontend receptions
grep -rn "onMessage\|data.action\|then(\|useQuery\|subscribe" src/ frontend/ app/

# === FLUX D — UI internal ===

# Data sources
grep -rn "useStore\|useState\|defineProps\|createStore\|useSelector" src/ frontend/

# Reactivity
grep -rn "ref(\|reactive(\|computed(\|watch(\|useEffect\|useMemo" src/ frontend/

# === FLUX E — Pipeline ===

# Steps
grep -rn "step\|phase\|stage\|task\|substep" pipeline/ workflow/

# Statuses
grep -rn "DONE\|FAIT\|FAILED\|EN_COURS\|PENDING\|status" pipeline/ state/

# === FLUX F — Config ===

# Config files
find . -name "*.yaml" -o -name "*.yml" -o -name "*.toml" -o -name "*.env" -o -name "settings.*"

# Consumers
grep -rn "load_config\|getenv\|config\[" src/ core/

# === FLUX G — Agent cognition ===

# Assembled context
grep -rn "system_prompt\|messages\|build_context\|max_tokens\|truncate" agents/ core/ src/

# Memory recall
grep -rn "retrieve\|recall\|embedding\|top_k\|similarity" agents/ memory/ core/

# Actions and feedback
grep -rn "tool_calls\|tool_use\|function_call\|tool_result\|max_iterations" agents/ tools/ core/

# Consolidation
grep -rn "save_memory\|persist\|consolidate\|upsert" agents/ memory/ core/

# === UNIVERSAL ===

# Search for anything
grep -rn "term" src/ core/ lib/ tests/ docs/
```

---

## Output-reading protocol — Rule #30

### Why this protocol exists

AI agents skim by default. They scan the first and last lines, guess the rest, and conclude "everything looks OK". This protocol FORBIDS that behavior.

### Absolute rules

1. **NEVER SUMMARIZE** an output without having read it line by line first.
2. **NEVER SAY** "it looks good", "everything seems correct", "the rest is similar".
3. **Every claim = one proof.** If you say "the method exists", cite the exact line with its number.
4. **Every count = an exhaustive tally.** If you say "3 errors", list the 3.
5. **If a result is 200 lines long, read 200 lines.** Split into segments if necessary. Skip nothing.

### The 5-pass protocol

**PASS 1 — Raw count**
- Total number of data lines (excluding decoration)
- Number of negative lines (NO, 0, None, null, error, failed, ABSENT, N/A, empty)
- Number of positive lines (OK, PASS, True, success)
- Write: `COUNT: X lines, Y negatives, Z positives`

**PASS 2 — Line-by-line reading**
For EACH line, in order:
- READ word by word
- IDENTIFY the type (value, path, status, name)
- Flag IMMEDIATELY: corrupted characters, truncated data, unexpected zeros, unexpected NULLs, duplicates, incoherences, broken encoding, suspicious default values

**PASS 3 — Cross-referencing**
- Cross-check columns/sections against each other
- Verify coherence of totals (sum of subtotals = total?)
- Identify absences (which data SHOULD be there but is NOT?)

**PASS 4 — Agent-lie detection**
Known patterns to verify:
- "DONE" without proof — no command confirms the action
- Embellished count — "9/10" but 3 are actually broken
- Masked error — `except: pass` swallowing errors
- Selective result — `| head -5` instead of showing everything
- Partial status — "Tests pass" without a full visible count
- Criterion redefinition — the agent changes the definition of "success"

**PASS 5 — Implications and actions**
For each problem: impact, priority, probable root cause, exact corrective action.

### Reading score

After the 5 passes:
- `SCORE: X/Y clean lines`
- `VERDICT: CLEAN | SUSPECT — VERIFY | CORRUPTED — FIX`
- If a single line is critical, the verdict CANNOT be CLEAN.

---

## Forbidden behaviors

These anti-patterns are the most frequent. They MUST be intercepted.

### Reading laziness
- **FORBIDDEN**: Reading 30 lines out of 500 and concluding
- **FORBIDDEN**: "The rest of the file follows the same pattern"
- **FORBIDDEN**: Opening a file, reading the class signature, and closing it
- **MANDATORY**: Read integrally or in segments without skipping

### Diagnosis by assumption
- **FORBIDDEN**: "This method should exist" → GREP first
- **FORBIDDEN**: "This module is probably initialized" → VERIFY first
- **FORBIDDEN**: "The import seems correct" → READ the real import
- **FORBIDDEN**: "The agent understood the instruction" → READ the context actually sent and the decision trace
- **MANDATORY**: Every stated fact is proven by an executed command

### Reactive instead of proactive debugging
- **FORBIDDEN**: Execute first, diagnose afterward when it crashes
- **MANDATORY**: Trace the complete flow BEFORE modifying (Debug Pilleur Master)
- **MANDATORY**: Understand the INPUT/OUTPUT/CONNECTIONS context before touching anything

### False diagnostics
- **FORBIDDEN**: "Everything works correctly" without execution proof
- **FORBIDDEN**: Marking "DONE" without having executed the verification command
- **FORBIDDEN**: Showing a partial result (head -5, tail -3) as proof of global success
- **MANDATORY**: Complete result or nothing. If too long, split and show everything.

### Excessive conservatism
- **FORBIDDEN**: Asking permission at every micro-step when the task is clear
- **FORBIDDEN**: Proposing instead of executing when the instruction is explicit
- **MANDATORY**: If the instruction says "do it", DO IT. If in blocking doubt, ONE SINGLE STOP, not a STOP every 3 files.

---

## How to use this document

### In a new project

1. Copy this file into `docs/DEBUG_PILLEUR_MASTER.md`
2. In the project's CLAUDE.md, add a reference:
   ```
   > Debug methodology: docs/DEBUG_PILLEUR_MASTER.md
   > This protocol is NON-NEGOTIABLE for every diagnostic.
   ```
3. Adapt the grep commands to the project's specific paths

### For every development task

1. Identify the flux type(s) the task touches (A to G)
2. Trace the existing chain link by link, with proof (file:line)
3. Write the target chain: every link to create, modify or connect
4. Implement link by link, in flux order
5. Fill in the checklist of every flux touched — every box proven by an executed command
6. Declare "DONE" only when the target chain is traced end to end

### For every bug or modification

1. Identify the relevant flux type(s) (A to G)
2. Follow the flux steps in order
3. Fill in the checklist — every point verified with proof
4. If a point is missing, the flux is broken → fix it BEFORE continuing

### To conclude any task

The agent doing the work is itself an agent: before concluding, it applies flux G to its own work.

1. Perception and attention: did I read the real context (files, outputs, instructions) or assume it?
2. Reasoning: does my decision come from what I read, or from habit?
3. Feedback: does every "DONE" I announce match a real observation?
4. Consolidation: is what I learned written where the next cycle will find it?

### For every result reading

1. Apply the 5-pass protocol
2. Produce the score and the verdict
3. NEVER conclude without having read integrally

---

*Original methodology: AVAA v3.3.0 session (2026-01-27)*
*Universal version: 2026-05-24 — English translation: 2026-07-02 — Flux G (agent cognition): 2026-10-26*
*Applicable to any software project — no dependency on any specific framework or language*
