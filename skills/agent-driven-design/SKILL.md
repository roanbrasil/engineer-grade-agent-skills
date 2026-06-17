---
name: agent-driven-design
description: Expert in Agent-Driven Design (ADD) — the Model/Harness decomposition framework for designing LLM-based agent systems. Use when designing agent architecture, deciding where logic belongs (Model vs Harness vs Infra), choosing agent topologies, building production observability, or diagnosing why an agent performs poorly.
---

# Agent-Driven Design (ADD)

A conceptual framework for designing systems where LLM-based agents are first-class architectural citizens.

## Core Thesis

**Every agent is composed of exactly two parts: a Model and a Harness. Neither alone is an agent.**

```
  ┌─────────────────────────────────────────────────────────────┐
  │                        AGENT                                │
  │                                                             │
  │  ┌──────────────────────┐  ┌──────────────────────────┐   │
  │  │        MODEL         │  │         HARNESS           │   │
  │  │                      │  │                           │   │
  │  │  The LLM             │  │  Everything else:         │   │
  │  │  Reasons             │  │  - Prompts & templates    │   │
  │  │  Generates           │  │  - Tool definitions       │   │
  │  │  Judges              │  │  - Memory management      │   │
  │  │  Plans               │  │  - Routing logic          │   │
  │  │                      │  │  - Validation             │   │
  │  │  (probabilistic)     │  │  - Retry & error handling │   │
  │  │                      │  │  - Context assembly       │   │
  │  └──────────────────────┘  └──────────────────────────┘   │
  │                                                             │
  │  Agent Context Boundary: what this agent knows, can        │
  │  do, and owns                                              │
  └─────────────────────────────────────────────────────────────┘
```

The Model is the engine. The Harness is the vehicle. You need both.

---

## The Model

The Model is the LLM — Claude, GPT-4, Gemini, or any generative model — responsible for:

- **Reasoning**: multi-step inference, planning, problem decomposition
- **Generation**: producing text, code, structured output
- **Judgment**: evaluating options, choosing between alternatives
- **Ambiguity resolution**: handling underspecified inputs by making reasonable inferences

**What the Model is NOT responsible for:**
- Knowing the current date/time (Harness provides this)
- Knowing which tools are available (Harness defines them)
- Managing conversation history (Harness manages context window)
- Retrying failed tool calls (Harness handles errors)
- Enforcing output formats (Harness validates and re-prompts)

The Model is the single most expensive component to change. Its capabilities and failure modes define the upper bound of what the agent can achieve.

---

## The Harness

The Harness is all code surrounding the Model. It is deterministic, testable, and fully under your control.

```
  ┌─────────────────────────────────────────────────────────────┐
  │                        HARNESS                              │
  │                                                             │
  │  Input Pipeline          Model Call           Output Pipeline
  │  ┌──────────────┐       ┌──────────┐         ┌───────────┐│
  │  │ Context      │       │          │         │ Parser    ││
  │  │ Assembly:    │──────►│  MODEL   │────────►│ Validator ││
  │  │ - System     │       │          │         │ Router    ││
  │  │   prompt     │       └──────────┘         └───────────┘│
  │  │ - Memory     │            ▲                      │      │
  │  │   retrieval  │            │ tool results         │      │
  │  │ - Tool defs  │       ┌────┴─────┐                │      │
  │  │ - History    │       │  Tool    │                ▼      │
  │  └──────────────┘       │  Executor│          Next Action  │
  │                         │  (MCP,   │          or Final     │
  │                         │  APIs,   │          Response     │
  │                         │  DBs)    │                       │
  │                         └──────────┘                       │
  └─────────────────────────────────────────────────────────────┘
```

**Harness responsibilities:**
- Assembling the context window before each Model call
- Defining and executing tools (via MCP or direct API calls)
- Managing conversation history and memory retrieval
- Parsing, validating, and routing Model output
- Handling errors, retries, and fallbacks
- Enforcing cost and latency budgets
- Emitting observability signals

---

## The Agent Context Boundary

Every agent has a Context Boundary — the conceptual scope of what it knows, can do, and owns.

```
  ┌──────────────────────────────────────────────────────────────┐
  │              AGENT CONTEXT BOUNDARY                          │
  │                                                              │
  │  Knows:                                                      │
  │  ├── Its role and instructions (system prompt)               │
  │  ├── Current conversation (context window)                   │
  │  ├── Retrieved knowledge (RAG / memory)                      │
  │  └── Tool results from this session                         │
  │                                                              │
  │  Can do:                                                     │
  │  ├── Tools it has been given (NOT all tools in the system)   │
  │  └── Actions within its defined scope                        │
  │                                                              │
  │  Owns:                                                       │
  │  ├── Its own state within a session                         │
  │  └── Its defined domain (not another agent's domain)        │
  │                                                              │
  │  Does NOT know:                                              │
  │  ├── What other agents are doing                            │
  │  ├── Secrets from other agent sessions                      │
  │  └── Tools not in its definition                            │
  └──────────────────────────────────────────────────────────────┘
```

A well-designed agent boundary prevents context leakage, limits blast radius of failures, and makes agents composable.

---

## The Decision Rules

### Where Does This Logic Belong?

```
  Is this logic about REASONING, JUDGMENT, or GENERATION?
  │
  ├─ YES → it belongs in the MODEL
  │        (let the LLM do it; don't try to code reasoning)
  │
  └─ NO → Is this logic about STRUCTURE, FLOW, or CONTRACT?
           │
           ├─ YES → it belongs in the HARNESS
           │        (deterministic code; fully testable)
           │
           └─ NO → Does this serve the Harness?
                    │
                    ├─ YES → it belongs in INFRA
                    │        (databases, queues, compute)
                    │
                    └─ NO → Re-examine the requirement
```

### Concrete Examples

| Logic | Belongs In | Why |
|---|---|---|
| "Is this code correct?" | Model | Requires reasoning and judgment |
| "Retry this tool call 3 times" | Harness | Deterministic control flow |
| "Which agent should handle this?" | Harness (router) | Structural routing decision |
| "What is the best approach?" | Model | Requires judgment |
| "Parse the JSON output" | Harness | Deterministic parsing |
| "Is this response safe?" | Model (or Harness rule) | Safety judgment; may need Model |
| "Store this in the database" | Infra (via Harness tool) | Infrastructure concern |
| "Format the prompt template" | Harness | Template rendering is deterministic |
| "Summarize this document" | Model | Generation task |

---

## The Improvement Rule

When an agent performs poorly, diagnose before you fix.

```
  Agent is not performing well
           │
           ▼
  Is the Model receiving CORRECT, COMPLETE context?
  Are the tools WELL-NAMED and CLEARLY DESCRIBED?
  Is the routing SENDING the right task to this agent?
           │
     ┌─────┴──────┐
     │ NO         │ YES
     ▼            ▼
  Harness-Driven    Is the Model reasoning INCORRECTLY
  Design (HDD)      despite correct context and tools?
                         │
                   ┌─────┴──────┐
                   │ YES        │ NO (it's still Harness)
                   ▼            ▼
              LLM-Driven     Go back, audit Harness
              Design (LLMDD) more carefully
```

### Harness-Driven Design (HDD)

Fix Harness problems first. Most agent failures are Harness failures.

**HDD interventions (in order of cost):**
1. **Improve the system prompt** — clearer role, clearer constraints, better examples
2. **Improve tool names and descriptions** — the Model picks tools by name; bad names = wrong tool calls
3. **Improve context assembly** — more relevant retrieved context; better history pruning
4. **Add output validation** — catch bad output formats; re-prompt instead of failing silently
5. **Fix routing** — ensure the right task goes to the right agent
6. **Add few-shot examples** — examples in the prompt are the highest-ROI intervention

### LLM-Driven Design (LLMDD)

Only after HDD is exhausted. LLMDD addresses Model-layer failures.

**LLMDD interventions (in order of cost):**
1. **Upgrade model tier** — use a more capable model (costly but fast to try)
2. **Chain-of-Thought prompting** — ask the model to reason step by step before answering
3. **Fine-tuning** — embed domain knowledge into the model weights (expensive; see Fine-tuning section)
4. **Model distillation** — use a large model to generate training data for a smaller, faster model

**Rule:** Never fine-tune before exhausting HDD. Fine-tuning is expensive, slow to iterate, and creates a model you must maintain. A better prompt is almost always cheaper.

---

## Agent Topology Patterns

### Single Agent

One model, one harness. Handles bounded, focused tasks.

```
  ┌─────────────────────────────────────────┐
  │              Single Agent               │
  │                                         │
  │   Input → [HARNESS] → [MODEL] → Output  │
  │                 ↑──────────┘            │
  │              (tool loop)                │
  └─────────────────────────────────────────┘
```

**Use when:**
- Task is well-defined and bounded
- No need to parallelize
- Simplicity and debuggability matter most
- Starting point for any new agent (always start here)

**When to graduate to multi-agent:**
- Task consistently exceeds context window
- Task has clearly separable subtasks with different expertise requirements
- Latency can be reduced by parallelization
- Task failure modes need isolation

---

### Orchestrator + Specialists

One orchestrator decomposes the goal. Specialists execute subtasks. Results aggregated by orchestrator.

```
  User Goal
      │
      ▼
  ┌─────────────────────────────────────────────────────────┐
  │                    Orchestrator                         │
  │  [HARNESS: task decomposer, result aggregator]          │
  │  [MODEL: decomposes goals, interprets results, plans]   │
  └───────────┬─────────────────┬───────────────────────────┘
              │ delegate         │ delegate
              ▼                  ▼
  ┌─────────────────┐    ┌─────────────────────┐
  │ Code Specialist │    │  Research Specialist │
  │ [HARNESS: code  │    │  [HARNESS: web tools,│
  │  tools, linter] │    │   doc retrieval]    │
  │ [MODEL: code    │    │  [MODEL: synthesis, │
  │  reasoning]     │    │   summarization]    │
  └─────────────────┘    └─────────────────────┘
```

**Strengths:** clear separation of concerns; specialists can be upgraded independently.

**Weaknesses:** orchestrator is a chokepoint; hard to trace failures across agent boundaries; orchestrator may accumulate too much responsibility.

**Anti-pattern — God Orchestrator:**
```
  # BAD: orchestrator does substantive work
  class Orchestrator:
      tools = [write_code, review_code, run_tests, deploy,
               search_web, query_db, send_email, ...]  # too many tools!

  # GOOD: orchestrator only decomposes and delegates
  class Orchestrator:
      tools = [delegate_to_agent, aggregate_results, replan]
```

---

### Pipeline

Output of Agent N is the input of Agent N+1. Sequential, predictable, easy to debug.

```
  Input
    │
    ▼
  ┌───────────┐
  │ Agent 1   │  (e.g., research)
  │ extractor │
  └─────┬─────┘
        │ structured output
        ▼
  ┌───────────┐
  │ Agent 2   │  (e.g., analysis)
  │ analyzer  │
  └─────┬─────┘
        │ analysis
        ▼
  ┌───────────┐
  │ Agent 3   │  (e.g., writing)
  │  writer   │
  └─────┬─────┘
        │
        ▼
      Output
```

**Use when:**
- Steps have clear input/output contracts
- Each step transforms the data in a predictable way
- Debuggability is important (inspect output between each stage)
- Tasks are inherently sequential (each step depends on prior)

**Caution:** pipeline failures cascade. Agent 2 cannot run if Agent 1 produces bad output. Add validation steps between agents, or use checkpointing.

---

### Parallel Fan-out

Same input dispatched to N agents simultaneously. Results aggregated.

```
              Input
                │
    ┌───────────┼────────────┐
    ▼           ▼            ▼
  Agent A    Agent B      Agent C
  (approach  (approach    (approach
    #1)        #2)          #3)
    │           │            │
    └───────────┼────────────┘
                ▼
           Aggregator
           (selects best,
            merges, or votes)
```

**Use when:**
- Multiple valid approaches to a problem; want to compare
- Redundancy for reliability (take first valid result = Speculative Execution)
- Independent subtasks that can run concurrently (speed)

**Aggregation strategies:**
- **Best-of-N**: have a judge agent select the best result
- **First-valid**: take whichever completes first and passes validation
- **Merge**: combine non-overlapping outputs (e.g., parallel research on different subtopics)
- **Vote**: majority vote on structured outputs (for classification, labeling)

---

### Reflection Loop

Agent output is fed back as critic input. Iterate until convergence or max rounds.

```
  Input
    │
    ▼
  ┌───────────────────────────────────────────────┐
  │                                               │
  │  ┌─────────────┐     ┌─────────────────┐     │
  │  │  Generator  │────►│    Critic       │     │
  │  │  Agent      │     │    Agent        │     │
  │  │             │◄────│  (evaluates,    │     │
  │  │  (improves  │     │   gives         │     │
  │  │   on        │     │   feedback)     │     │
  │  │   feedback) │     │                 │     │
  │  └─────────────┘     └────────┬────────┘     │
  │                               │               │
  │                         "Acceptable"?         │
  │                        ┌──────┴───────┐       │
  │                        │ NO           │ YES   │
  │                        └──────────────┘       │
  │                          (loop again)   Output│
  └───────────────────────────────────────────────┘
```

**Use when:**
- Quality matters more than speed (e.g., code generation, writing)
- Generator and critic can be the same model with different prompts
- Convergence criterion is definable (e.g., "no errors found", "score > 8")

**Termination conditions (required — prevent infinite loops):**
- Max iterations reached (hard stop)
- Critic scores output above threshold
- Critic finds no further improvements
- Diff between iterations falls below threshold

**Implementation:**
```python
async def reflection_loop(
    generator: Agent,
    critic: Agent,
    input: str,
    max_rounds: int = 3
) -> str:
    draft = await generator.run(input)
    for round in range(max_rounds):
        critique = await critic.evaluate(draft)
        if critique.score >= 8.0 or critique.no_issues:
            break
        draft = await generator.improve(draft, critique.feedback)
    return draft
```

---

## Harness Patterns

### RAG as a Harness Pattern

Retrieval-Augmented Generation is a Harness pattern, not a Model feature. The Harness retrieves; the Model reasons on what was retrieved.

```
  User Query
      │
      ▼ (Harness: retrieval pipeline)
  ┌─────────────────────────────────────────────────────┐
  │  1. Embed query                                     │
  │  2. Search vector store                             │
  │  3. Re-rank results                                 │
  │  4. Select top-K by relevance + diversity           │
  │  5. Assemble into context window                    │
  └─────────────────────────────────────────────────────┘
      │ context: [retrieved chunks] + query
      ▼ (Model: reasoning on retrieved context)
  ┌─────────────────────────────────────────────────────┐
  │  MODEL                                              │
  │  "Based on the retrieved context, the answer is..." │
  └─────────────────────────────────────────────────────┘
```

**Harness controls in RAG:**
- Embedding model selection
- Chunk size and overlap
- Number of retrieved chunks (top-K)
- Re-ranking strategy
- Hybrid search (semantic + keyword)
- Context assembly order (most relevant first/last)

**RAG failures are Harness failures:**
- "The model doesn't know X" → check if X is in the store; check retrieval quality
- "The model hallucinates despite having the document" → check chunk quality; check if relevant chunk is being retrieved
- "The model ignores retrieved context" → check context assembly; retrieved context may be buried

---

### Tool Design

Tools are Harness. Well-designed tools reduce the prompt complexity needed to use them.

**Tool design principles:**

1. **Name tools by what they do, not how they work**
   ```
   # BAD
   tools = ["execute_sql_query", "make_http_request", "read_filesystem"]

   # GOOD
   tools = ["search_orders", "get_customer_profile", "list_recent_transactions"]
   ```

2. **One tool, one purpose** — tools that do two things lead to ambiguous tool calls

3. **Return structured, typed output** — models parse tool results; structured output reduces parsing errors

4. **Include examples in tool descriptions**
   ```python
   Tool(
       name="search_orders",
       description="""Search customer orders by criteria.
       Examples:
       - search_orders(customer_id="C123") → orders for customer C123
       - search_orders(status="pending", after_date="2024-01-01") → pending orders since Jan 2024
       Returns: list of Order objects with id, status, total, items.""",
       ...
   )
   ```

5. **Fail loudly and descriptively** — tool errors become part of Model context; vague errors lead to Model confusion
   ```python
   # BAD: raises generic exception
   raise Exception("Database error")

   # GOOD: returns structured error the Model can reason on
   return ToolError(
       code="ORDER_NOT_FOUND",
       message="No order found with id='ORD-999'",
       suggestion="Try searching by customer_id instead"
   )
   ```

---

### Memory Types

```
  ┌─────────────────────────────────────────────────────────────┐
  │                      AGENT MEMORY                          │
  │                                                             │
  │  ┌─────────────────────────────────────────────────────┐   │
  │  │ WORKING MEMORY (context window)                     │   │
  │  │ Current conversation + active task state            │   │
  │  │ Scope: current session only                         │   │
  │  │ Managed by: Harness (context assembly)              │   │
  │  └─────────────────────────────────────────────────────┘   │
  │                                                             │
  │  ┌─────────────────────────────────────────────────────┐   │
  │  │ EPISODIC MEMORY (conversation history)              │   │
  │  │ Past conversations, decisions, outcomes             │   │
  │  │ Scope: across sessions; same user or same task      │   │
  │  │ Managed by: Harness (DB read on session start)      │   │
  │  └─────────────────────────────────────────────────────┘   │
  │                                                             │
  │  ┌─────────────────────────────────────────────────────┐   │
  │  │ SEMANTIC MEMORY (knowledge base)                    │   │
  │  │ Domain knowledge, facts, documentation              │   │
  │  │ Scope: global; shared across agents                 │   │
  │  │ Managed by: Harness (vector DB retrieval)           │   │
  │  └─────────────────────────────────────────────────────┘   │
  │                                                             │
  │  ┌─────────────────────────────────────────────────────┐   │
  │  │ PROCEDURAL MEMORY (learned patterns)                │   │
  │  │ How to solve classes of problems                    │   │
  │  │ Scope: embedded in system prompt or fine-tuned      │   │
  │  │ Managed by: Harness (prompt) or Model (fine-tuning) │   │
  │  └─────────────────────────────────────────────────────┘   │
  └─────────────────────────────────────────────────────────────┘
```

---

### Loop Patterns

All agent loop patterns are Harness designs. The Model generates within each step; the Harness controls iteration.

#### ReAct (Reason + Act)

```
  Input
    │
    ▼
  Thought: [Model reasons about what to do next]
    │
    ▼
  Action: [Model picks a tool]
    │
    ▼
  Observation: [Harness executes tool, returns result]
    │
    ▼
  Thought: [Model reasons on observation]
    │
    ▼
  ... (repeat until Answer)
    │
    ▼
  Answer: [Model generates final response]
```

**Harness role in ReAct:** detect when Model outputs a tool call; execute the tool; inject observation back into context; loop until Model outputs a final answer.

#### Plan-Execute

```
  Input
    │
    ▼
  [MODEL: generate explicit plan]
  Plan: ["Step 1: search X", "Step 2: analyze Y", "Step 3: write Z"]
    │
    ▼ (Harness iterates over plan steps)
  Execute Step 1 → result_1
  Execute Step 2 → result_2 (using result_1)
  Execute Step 3 → result_3 (using result_1, result_2)
    │
    ▼
  Final Output
```

**Advantage over ReAct:** plan is visible and auditable before execution. Harness can validate the plan, insert human approval, or modify steps before execution begins.

#### Reflection

See Reflection Loop topology above. In the loop pattern framing: one Model can play both generator and critic with different system prompts, controlled by the Harness.

```python
# Harness controls the loop; Model plays both roles
async def reflection_loop(draft: str, context: str) -> str:
    for _ in range(MAX_ROUNDS):
        # Model as critic
        critique = await model.call(
            system=CRITIC_SYSTEM_PROMPT,
            messages=[{"role": "user", "content": f"Critique:\n{draft}"}]
        )
        if is_acceptable(critique):
            return draft
        # Model as generator
        draft = await model.call(
            system=GENERATOR_SYSTEM_PROMPT,
            messages=[{"role": "user", "content":
                f"Improve this based on feedback:\n{draft}\n\nFeedback:\n{critique}"}]
        )
    return draft
```

---

## Production Concerns

### Evals: Three Scopes

```
  ┌─────────────────────────────────────────────────────────────┐
  │                                                             │
  │  MODEL EVAL                                                 │
  │  Scope: the LLM in isolation                               │
  │  Measures: reasoning quality, knowledge, format adherence  │
  │  How: benchmark datasets, unit-test-style prompts          │
  │  When: evaluating model upgrade/downgrade                  │
  │                                                             │
  │  AGENT EVAL (Harness + Model together)                     │
  │  Scope: one agent with its Harness                         │
  │  Measures: task completion rate, tool call correctness     │
  │  How: end-to-end task scenarios; compare output to golden  │
  │  When: after Harness changes, before deploying             │
  │                                                             │
  │  SYSTEM EVAL (multi-agent)                                 │
  │  Scope: the full multi-agent system                        │
  │  Measures: goal completion, latency, cost, error rate      │
  │  How: representative user goals; human evaluation          │
  │  When: after topology changes; production monitoring       │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘
```

**Eval anti-patterns:**
- Running system evals when you have a Model eval problem (hard to isolate)
- No automated evals (changes break things silently)
- Eval datasets that don't reflect production distribution
- Using the same model being evaluated to judge its own output (self-grading bias)

---

### Observability

The Harness must expose signals. The Model is a black box; the Harness is your instrumentation layer.

```
  What to emit from the Harness:

  ┌──────────────────────────────────────────────────────────┐
  │ Per Agent Call:                                          │
  │   - agent_id, task_id, session_id (correlation)         │
  │   - model_id, model_version (what Model was called)     │
  │   - input_tokens, output_tokens (cost tracking)         │
  │   - latency_ms (end-to-end and per model call)          │
  │   - tool_calls: [{name, args, result, latency}]         │
  │   - success: bool, error_type if failed                 │
  │                                                          │
  │ Per Tool Call:                                           │
  │   - tool_name, tool_args (what was called)              │
  │   - result_size_tokens (how much context it added)      │
  │   - latency_ms (tool execution time)                    │
  │   - success: bool, error if failed                      │
  │                                                          │
  │ Per Session:                                             │
  │   - turns_count (conversation length)                   │
  │   - total_cost_usd                                      │
  │   - goal_completed: bool                                │
  │   - human_interventions (HITL events)                   │
  └──────────────────────────────────────────────────────────┘
```

**Structured logging pattern:**
```python
@contextmanager
def trace_agent_call(agent_id: str, task: str):
    span = tracer.start_span("agent_call")
    span.set_attributes({
        "agent.id": agent_id,
        "agent.task": task[:200],  # truncate for index
    })
    try:
        yield span
        span.set_attribute("agent.success", True)
    except Exception as e:
        span.set_attribute("agent.success", False)
        span.set_attribute("agent.error", str(e))
        raise
    finally:
        span.end()
```

**Key metrics to alert on:**
- Token usage spike (context bloat; runaway loops)
- Tool error rate rise (external dependency issues)
- Agent latency p99 increase (model slowdown or context growth)
- Goal completion rate drop (Harness or Model regression)

---

### Fine-tuning vs. Prompting

```
  Should I fine-tune?
          │
          ▼
  Have I exhausted all Harness-Driven Design options?
  (system prompt, tool descriptions, context assembly,
   few-shot examples, output validation, re-prompting)
          │
     ┌────┴─────┐
     │ NO       │ YES
     ▼          ▼
  Do HDD      Is the problem consistent and
  first.      reproducible? (not stochastic)
              │
         ┌────┴──────┐
         │ NO        │ YES
         ▼           ▼
   HDD is still  Do you have 100+ high-quality
   the answer.   (input, ideal_output) examples?
                 │
            ┌────┴──────┐
            │ NO        │ YES
            ▼           ▼
      Collect data  Fine-tune may be
      first.        appropriate.
```

**Fine-tune to embed into Model weights:**
- Domain-specific format compliance (e.g., always output JSON in specific schema)
- Persona/style consistency at scale
- Latency reduction via distillation (replace large model with fine-tuned small model)

**Never fine-tune to:**
- Inject knowledge that changes frequently (use RAG)
- Fix a bug in the system prompt (fix the prompt)
- Paper over a Harness problem (fix the Harness)

---

### Cost and Latency

All cost and latency decisions are Harness decisions.

```
  Cost = Σ (input_tokens × input_price + output_tokens × output_price)
         across all model calls in the agent session

  Levers (all Harness):
  ┌─────────────────────────────────────────────────────────────┐
  │ MODEL TIER SELECTION                                        │
  │ Use the cheapest model that achieves acceptable quality     │
  │ Route simple tasks to smaller models                        │
  │ Route complex tasks to larger models                        │
  │                                                             │
  │ CONTEXT COMPRESSION                                         │
  │ Fewer input tokens = lower cost                             │
  │ Summarize history; prune irrelevant context                 │
  │                                                             │
  │ CACHING                                                     │
  │ Prompt caching: same system prompt across calls             │
  │ Semantic caching: cache outputs for similar inputs          │
  │ Tool result caching: don't re-fetch unchanged data          │
  │                                                             │
  │ BATCHING                                                    │
  │ Batch independent subtasks into single model call           │
  │ (only when subtasks don't need intermediate results)        │
  │                                                             │
  │ PARALLELISM                                                 │
  │ Fan-out reduces wall-clock time; same total cost            │
  │ Use for latency-critical, cost-tolerant workloads           │
  └─────────────────────────────────────────────────────────────┘
```

**Routing by model tier (Harness router):**
```python
class ModelRouter:
    def select_model(self, task: Task) -> str:
        if task.complexity == "simple" and task.type in ["classify", "extract"]:
            return "claude-haiku-3"       # fast, cheap
        if task.type in ["code_generation", "analysis"]:
            return "claude-sonnet-4"      # balanced
        if task.requires_deep_reasoning:
            return "claude-opus-4"        # most capable
        return "claude-sonnet-4"          # safe default
```

---

## Anti-Patterns

### Anti-Pattern 1: Logic That Belongs in Harness Is Put in Model

**Symptom:** System prompt contains logic like "if the user asks about billing, respond with X; if they ask about technical issues, respond with Y; if they ask about both..."

**Problem:** You're trying to make the Model do routing via prompt engineering. This is fragile, expensive (routing logic burns tokens on every call), and untestable.

**Fix:**
```python
# BAD: routing via system prompt
system = """
If the user asks about billing, respond as a billing specialist.
If the user asks about technical issues, respond as a tech support agent.
If the user asks about account management, respond as...
"""

# GOOD: routing in Harness, specialist agents with focused prompts
intent = await router.classify(user_message)
agent = registry.get_agent(intent)
response = await agent.run(user_message)
```

---

### Anti-Pattern 2: Logic That Belongs in Model Is Put in Harness

**Symptom:** Complex if/else chains in Harness code trying to handle all cases of agent reasoning.

```python
# BAD: Harness trying to reason
if "error" in tool_result.lower():
    if "not found" in tool_result.lower():
        next_action = "search_by_id"
    elif "permission" in tool_result.lower():
        next_action = "request_access"
    elif "timeout" in tool_result.lower():
        next_action = "retry"
    # ... 50 more cases
```

**Problem:** you can't enumerate all cases. The Model is better at this.

**Fix:** give the Model the tool result and let it reason about the next action. The Harness executes whatever the Model decides.

---

### Anti-Pattern 3: Context Bloat

**Symptom:** Agent quality degrades over long conversations. Cost per turn keeps increasing. Token usage steadily climbs.

**Problem:** the context window grows unbounded. Irrelevant early context crowds out relevant recent context. "Lost in the middle" — models attend poorly to context in the middle of a long window.

**Fix:**
```python
# BAD: append everything
messages.append(new_message)
messages.append(tool_result)
# context grows forever

# GOOD: managed context window
messages = context_manager.add(new_message)
# context_manager prunes old turns, summarizes when over budget
# always keeps: system + recent N turns + current message
```

---

### Anti-Pattern 4: Monolithic Agent

**Symptom:** One agent handles all tasks for all users. System prompt is 10,000 tokens. Agent has 40 tools. Performance is inconsistent across task types.

**Problem:** the agent is trying to be everything. Long system prompts dilute focus. Many tools confuse tool selection. Different task types have conflicting requirements.

**Fix:** decompose into specialized agents. Use an orchestrator to route. Each specialist has a focused system prompt and a small, relevant tool set.

```
  # BAD: one agent, all tasks
  MasterAgent(
      system_prompt=10000_token_monolith,
      tools=[all_40_tools]
  )

  # GOOD: specialized agents
  Orchestrator(tools=[route, delegate, aggregate])
  → CodeAgent(system=focused_on_code, tools=[read, write, run])
  → DataAgent(system=focused_on_data, tools=[query, analyze])
  → WriteAgent(system=focused_on_writing, tools=[search, draft])
```

---

### Anti-Pattern 5: No Agent Boundary

**Symptom:** Agents share a single context window or share memory indiscriminately. Agent A's private reasoning is visible to Agent B. Debugging is impossible because you can't tell which agent produced an output.

**Fix:** enforce Agent Context Boundaries. Each agent gets its own context. Use explicit message passing to share results. Use shared context stores (Blackboard) only for intended shared state.

---

## ADD Conceptual Layer Map

```
  ┌─────────────────────────────────────────────────────────────┐
  │                   USER / BUSINESS GOAL                      │
  └───────────────────────────┬─────────────────────────────────┘
                              │
  ┌───────────────────────────▼─────────────────────────────────┐
  │                 AGENT TOPOLOGY LAYER                        │
  │  (Single / Orchestrator+Specialists / Pipeline /            │
  │   Fan-out / Reflection Loop)                               │
  └───────────────────────────┬─────────────────────────────────┘
                              │
  ┌───────────────────────────▼─────────────────────────────────┐
  │                 INDIVIDUAL AGENT LAYER                      │
  │  ┌─────────────────────┐  ┌─────────────────────────────┐  │
  │  │     HARNESS         │  │          MODEL               │  │
  │  │  Prompts, Tools,    │  │   LLM reasoning,             │  │
  │  │  Memory, Routing,   │  │   generation,                │  │
  │  │  Validation, Loops  │  │   judgment                   │  │
  │  └─────────────────────┘  └─────────────────────────────┘  │
  └───────────────────────────┬─────────────────────────────────┘
                              │
  ┌───────────────────────────▼─────────────────────────────────┐
  │                  INTEGRATION LAYER                          │
  │  MCP (vertical: tools/resources)                            │
  │  A2A (horizontal: agent-to-agent)                           │
  └───────────────────────────┬─────────────────────────────────┘
                              │
  ┌───────────────────────────▼─────────────────────────────────┐
  │                  INFRASTRUCTURE LAYER                       │
  │  Vector stores, relational DBs, message queues,             │
  │  compute, observability, secret management                  │
  └─────────────────────────────────────────────────────────────┘
```

ADD provides the vocabulary to reason about any layer, assign responsibility correctly, and make upgrade decisions without rewriting the whole system.
