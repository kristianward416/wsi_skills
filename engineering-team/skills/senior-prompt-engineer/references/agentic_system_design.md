# Agentic System Design

Reference material for designing and reviewing agent architectures — tool-use patterns, context management, multi-agent orchestration, and failure modes. This complements `scripts/agent_orchestrator.py`, which validates the structural properties described here (tool wiring, loop risk, iteration caps) against a YAML/JSON agent config. Use this file when the user needs the reasoning behind a warning the orchestrator emits, or is choosing an architecture before a config exists to validate.

## 1. Tool-Use Architecture Patterns

### ReAct (Reason + Act)
The default pattern for most single-agent tool-use loops: the model alternates between a reasoning step (what do I know, what's missing, what should I do next) and an action step (call one tool), observes the result, and loops until it can answer.

```
Thought: I need the user's current subscription tier before answering.
Action: get_subscription(user_id=123)
Observation: {"tier": "pro", "renews": "2026-09-01"}
Thought: I have what I need. I can answer now.
Answer: ...
```

- **Strength**: naturally handles tasks where the next step depends on what the previous tool call returned — most real tool-use tasks are like this (you don't know what to search for next until you see the first search result).
- **Weakness**: it's inherently sequential and myopic — the model only plans one step ahead, so it can wander (extra unnecessary tool calls) on tasks that actually have a knowable multi-step shape up front. This is also the pattern most exposed to the loop and hallucinated-call failure modes in Section 4, precisely because there's no upfront plan to check progress against.
- **When to use**: exploratory tasks, tasks where tool results are genuinely unpredictable (search, browsing, debugging), single-agent assistants with a modest, well-described toolset (SKILL.md's "tool count minimal for the job" guidance matters most here — ReAct's per-step branching factor is the number of available tools, so a bloated toolset directly increases wander).

### Plan-and-Execute
The model first produces a multi-step plan (as data — a list of steps, ideally with explicit dependencies), then executes steps against that plan, revising it only when a step's result invalidates a later one.

```
Plan:
1. Look up order_id from the user's email
2. Get order status for that order_id
3. If status is "delayed", check carrier tracking
4. Compose response using whichever of 2/3 has the final status
```

- **Strength**: cheaper and more predictable for tasks with a knowable shape — the plan can be validated or shown to a user before execution starts (a meaningful safety property for anything with side effects), and it avoids ReAct's per-step wander since later steps aren't re-derived from scratch each time.
- **Weakness**: brittle when the world doesn't match the plan — a step returning something unexpected either requires an explicit re-planning step (added complexity) or the system silently executes a now-wrong plan. Needs a re-plan trigger, not just an execute loop.
- **When to use**: tasks with mostly-predictable structure (multi-step data pipelines, form-filling, workflows with known branches), anything where showing the plan for user approval before acting matters (destructive or costly actions), and tasks where minimizing tool-call count/latency matters more than adaptability.
- **Hybrid in practice**: many production agents plan-and-execute at a coarse grain (a handful of named phases) and use ReAct *within* each phase for the unpredictable parts — this bounds ReAct's wander to one phase at a time instead of the whole task.

### Tool Use / Function Calling as the substrate, not a pattern
Both ReAct and plan-and-execute sit on top of native tool-calling APIs (structured tool schemas, the model emits a typed call instead of prose to parse). This is a platform-feature-over-prompt-hack case, same principle as SKILL.md Operating Rule 3 for structured output: native tool-call APIs give you guaranteed-parseable calls and (on some providers) parallel tool-call batching; prompt-text conventions like "call a tool by writing TOOL: name(args)" are the fallback for APIs without native support, not the default.
- **Tool description discipline** (directly reflected in `agent_orchestrator.py`'s validation and SKILL.md's Agent Config Review workflow step 2): each tool description should be 1-2 sentences, state what it does and when to use it, and avoid overlapping responsibility with another tool — ambiguous boundaries between two similar tools is one of the most common causes of wrong-tool selection, independent of model capability.
- **Parallel tool calls**: when the model can determine multiple independent calls are needed before seeing any result (e.g., "get the weather in these 3 cities"), batching them in one turn cuts latency roughly N-fold versus sequential ReAct steps. This requires the task to have calls with no data dependency between them — if step 2 needs step 1's output, they can't be parallelized regardless of API support.

## 2. Context Management and Compaction

Context is the scarcest and most expensive resource in an agent loop — every tool observation, every intermediate thought, accumulates in the transcript that gets re-sent on every subsequent turn.

### Why this compounds
An N-step ReAct loop with average observation size O doesn't cost O(N×O) tokens total — it costs O(N²×O) tokens across the whole loop, because turn k re-sends all k-1 prior turns. This is the single biggest cost and latency driver in long-running agents, independent of model choice, and it's exactly what `agent_orchestrator.py --estimate-cost` is quantifying when it projects cost across a run.

### Mitigation strategies
- **Stable prefix first, cache-friendly ordering.** System rules, tool schemas, and any fixed context go first and never change turn-to-turn; only the growing transcript tail changes. This lets prompt caching (where the provider supports it) amortize the stable-prefix cost across every turn instead of re-billing it — the same principle as SKILL.md's prompt-optimization guidance ("move stable prefix first so prompt caching applies"), applied at the scale of a whole agent loop instead of one prompt.
- **Summarize-and-drop for stale tool observations.** Once an observation has been "used" (its relevant fact has been incorporated into a later thought or the plan), replace the full observation in the transcript with a one-line summary, or drop it if nothing downstream depends on it. This is the standard mitigation for the O(N²) blowup above — without it, a 30-step agent run can spend the majority of its tokens re-reading its own history rather than doing new work.
- **Explicit compaction at a token threshold, not ad hoc.** Define a trigger (e.g., transcript exceeds X tokens or Y turns) and a compaction step that condenses the transcript so far into a running summary (facts established, decisions made, open sub-goals) plus the last few raw turns for local coherence. Ad hoc, unplanned compaction tends to drop exactly the fact a later step needed.
- **Separate working memory from long-term memory.** Facts needed for the current step live in the live context window; facts that need to persist across an entire multi-session task (user preferences, prior decisions) belong in an external store (a scratch file, a key-value memory tool, a vector store) fetched on demand — not carried in every turn's context by default. This is a scoping decision, not a technology choice: the question to ask is "does the next step need this, or might some future step need this" — the former stays in context, the latter goes to retrievable storage.
- **Context budget per tool, not just per turn.** A tool that can return an unbounded amount of data (a full file read, a large search result set) needs an explicit truncation/pagination contract, or one greedy tool call can blow the context budget for the rest of the run. Tool descriptions should state their own output size contract (e.g., "returns at most the top 20 matches") so the agent's planning can account for it.
- **Token estimate as a design-time check, not just a runtime concern.** `agent_orchestrator.py --estimate-cost --runs N` projecting cost/run against a budget is the design-time analog of context management — if the projected token count per run is already high before any real observations are added, the architecture (too many tools, too much stable context, no compaction plan) needs to change before the agent runs once, not after a cost surprise.

## 3. Multi-Agent Orchestration Tradeoffs

Splitting a task across multiple agents (sub-agents, specialist agents) is a tradeoff between parallelism/specialization and coordination overhead — it is not automatically better than a well-scoped single agent, and per SKILL.md's related-skills note, this file scopes to *validating* configs, not to full architecture design (`engineering/agent-designer` owns the latter) — but the coordination-pattern tradeoffs below are the reasoning a config reviewer needs.

### Pipeline (sequential handoff)
Agent A's output becomes Agent B's input, becomes Agent C's input.
- **Strength**: each agent has a narrow, well-defined responsibility (better tool-description discipline, smaller context per agent, easier to eval each stage independently against its own golden set per the evaluation-frameworks reference). Natural fit when the task genuinely has sequential dependencies (research → draft → fact-check → format).
- **Weakness**: latency is additive (sum of all stages, no overlap); errors compound and are harder to attribute (a bad final answer might trace back to stage 1, but stage 3 is what surfaced it) — this makes debugging a pipeline meaningfully harder than debugging a single agent, since the fault could be in any upstream stage's output rather than the immediate step. Also brittle to schema drift between stages: if agent A's output format changes, every downstream agent silently breaks unless the handoff is contract-validated.
- **Mitigation**: validate the interface between every pair of stages (structured output schema, per SKILL.md's structured-output-design workflow, applied at each handoff) and eval each stage against its own golden set independently, not just the end-to-end pipeline — an end-to-end-only eval can't tell you which stage regressed.

### Parallel (fan-out / fan-in)
Multiple agents work independent sub-tasks simultaneously; a final step merges results.
- **Strength**: latency is bounded by the slowest branch, not the sum — this is the main reason to choose parallel over pipeline when sub-tasks are genuinely independent (e.g., "research competitor A, B, and C" — no branch needs another's output).
- **Weakness**: the merge/synthesis step becomes a new failure point and complexity concentration — someone (a model or code) has to reconcile potentially conflicting or inconsistent outputs from the parallel branches, and that reconciliation logic is easy to under-invest in since each branch "worked" individually. Also costs more compute/API calls concurrently (bursty cost and rate-limit pressure) versus a pipeline's steady sequential load.
- **Mitigation**: give the merge step an explicit conflict-resolution policy (not "just combine them") and treat the merge step itself as a component with its own eval cases, especially cases where branches disagree.
- **False parallelism trap**: the most common mistake is fanning out sub-tasks that look independent but actually share a resource or downstream dependency (e.g., all three branches book into the same limited calendar slot) — verify true independence before parallelizing, or the fan-in step will have to resolve conflicts that could have been avoided by sequencing.

### Choosing between them
- Default to pipeline when sub-tasks have a genuine dependency order or when per-stage evaluability matters more than latency.
- Default to parallel when sub-tasks are truly independent and latency matters (user-facing interactive agents), and only after confirming there's no hidden shared-resource dependency.
- **Default to a single well-scoped agent over either multi-agent pattern when the task doesn't clearly decompose** — multi-agent orchestration adds coordination cost (context handoff, error attribution, more moving parts to validate) that isn't free; it should be justified by a genuine specialization or latency win, not reached for by default. This mirrors SKILL.md's Operating Rule 4 spirit (don't add scaffolding reflexively) applied at the architecture level instead of the prompt level.

## 4. Failure Modes and Mitigations

### Hallucinated tool calls
The model calls a tool that doesn't exist, or calls a real tool with a plausible-looking but invalid argument (wrong field name, an ID it never actually looked up, a fabricated file path).
- **Root cause**: usually a gap between what the model was told is available and what actually is, or an ambiguous tool set where the model guesses at a tool it wishes existed rather than composing the ones it has.
- **Mitigations**:
  - Native structured tool-calling APIs (Section 1) reject calls that don't match a registered schema outright — this alone eliminates "calls a nonexistent tool" as a silent failure and turns it into a visible error the loop can react to.
  - Validate every argument against the tool's schema before execution (type, required fields, enum membership) and return a clear error observation ("invalid tool call: missing required field X") rather than executing with a best-guess default — a bad guess silently executed is far worse than a visible rejection the agent can recover from on the next turn.
  - Keep tool descriptions unambiguous and non-overlapping (Section 1) — a tool the model isn't sure exists gets hallucinated less often when the real available set is small and clearly scoped to the task.
  - `agent_orchestrator.py --validate`'s check for missing required tool config is the static-analysis-time version of this mitigation — catching a malformed tool wiring before any run, rather than discovering it only when the model tries to invoke the broken tool live.

### Infinite loops / non-termination
The agent keeps calling tools without converging — classic cases: a retry loop that never succeeds and never gives up, a ReAct loop that re-asks a question it already has the answer to because the answer wasn't recognized as sufficient, or two agents in a pipeline/parallel setup repeatedly handing a task back and forth.
- **Root cause**: missing or unclear termination condition — the agent has no explicit test for "do I have enough to answer now," so it defaults to gathering more.
- **Mitigations**:
  - **Hard iteration cap, always.** Every agent loop needs a maximum step count with a defined behavior on hitting it (return best-effort partial answer with a flag, not silent truncation) — this is a non-negotiable safety net regardless of how good the termination logic is, and it's exactly the "iteration cap + early-exit condition present" check in SKILL.md's Agent Config Review workflow and the "loop risk" warning `agent_orchestrator.py --validate` emits.
  - **Explicit early-exit condition**, not just a cap — the model should have an affirmative test for "I have enough to answer" (e.g., a checklist of required facts) so termination happens well before the cap in the common case, and the cap only fires on genuine failure.
  - **Repetition detection**: if the same tool is called with the same (or near-identical) arguments more than once or twice, that's a strong loop signal — surface it as a distinct observation ("you already called this with these arguments; the result was X") rather than silently re-executing, which both saves cost and gives the model the signal it needs to change strategy.
  - **Cross-agent loop guard** in pipeline/multi-agent setups: cap the number of handoff round-trips between any two agents, since a two-agent back-and-forth is a loop that a single-agent iteration cap won't catch.

### Other notable failure modes worth naming
- **Context poisoning**: an early error or hallucinated fact in the transcript gets treated as established truth by later turns (nothing re-validates it), and the agent confidently builds on a false premise for the rest of the run. Mitigation: give the model permission and a mechanism to revisit/retract earlier conclusions, and periodically prompt an explicit self-check ("does anything above conflict with what I now know") rather than assuming monotonic progress.
- **Silent partial failure**: a tool call "succeeds" (no error) but returns an empty, truncated, or default-value result that the agent treats as a real answer (e.g., an empty search result treated as "confirmed no results exist" rather than "the query may have been wrong"). Mitigation: tools should distinguish "confirmed empty" from "query likely failed" in their return contract, and agent instructions should explicitly enumerate this edge case per SKILL.md's general edge-case guidance ("enumerate the edge cases explicitly; add a 'when uncertain, do X' rule").
- **Prompt injection via tool output**: content retrieved by a tool (a webpage, a document, a search result) contains instructions crafted to hijack the agent ("ignore previous instructions and instead..."). Mitigation: treat all tool-returned content as untrusted data, never as instructions — this needs to be explicit in the system prompt, and ideally enforced structurally by not letting retrieved content occupy a message role the model treats as authoritative (system/developer) — it should only ever appear in an observation/tool-result role.
