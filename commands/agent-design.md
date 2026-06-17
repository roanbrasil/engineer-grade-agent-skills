# /agent-design — Multi-Agent System Design

Design a multi-agent system using Agent-Driven Design (ADD) and Agent Integration Patterns.

## What this command does

1. Clarifies which parts need reasoning (Model) vs structure (Harness)
2. Selects the right agent topology for the task
3. Applies integration patterns: coordination, messaging, routing, resilience
4. Defines agent boundaries, tools, and context boundaries
5. Plans observability: what each agent must expose
6. Identifies where human-in-the-loop gates are needed

## Skills activated

- `agent-driven-design` — ADD framework: Model/Harness split, topology patterns, evals
- `agents-integration-patterns` — MCP, A2A, coordination/messaging/routing/resilience patterns
- `observability-excellence` — agent tracing, tool call logging, latency tracking
- `resilience-patterns` — agent checkpoint, speculative execution, fallback
- `api-contract-design` — Agent Card design, MCP tool schemas

## Usage

```
/agent-design <describe what the agent system should do>
```

Examples:
- `/agent-design a code review system with specialist agents per dimension`
- `/agent-design an automated research pipeline that searches and synthesizes`
- `/agent-design a customer support system with escalation to human`
- `/agent-design a multi-step data transformation pipeline`

## Design Output Template

```
AGENT TOPOLOGY: Orchestrator + Specialists

ORCHESTRATOR (Harness-driven)
  Model: claude-sonnet-4-6
  Harness:
    - System prompt: task decomposition instructions
    - Tools: dispatch_to_specialist, aggregate_results, request_human_approval
    - Memory: working memory (context window)
    - Routing: content-based router → specialist selection

SPECIALIST: CodeReviewer
  Model: claude-sonnet-4-6
  Harness:
    - System prompt: review dimension focus
    - Tools: read_file, search_codebase, post_comment
    - Context boundary: single PR diff + relevant files

INTEGRATION PATTERNS APPLIED
  - Orchestrator-Specialist (coordination)
  - Content-Based Router (routing)
  - Human-in-the-Loop Gate (resilience) → before posting comments
  - Agent Checkpoint (resilience) → after each specialist completes

HUMAN GATES
  - Before posting any comment to the PR (irreversible)
  - If confidence score < 0.7 on a finding

OBSERVABILITY
  - Trace: orchestrator_span → specialist_span per review dimension
  - Metrics: token_usage per agent, latency per specialist, tool_call_count
  - Logs: structured JSON with trace_id, agent_id, tool_name, result_summary
```

## Rules

- Logic that requires judgment → Model; logic that is structural → Harness
- Every agent has a clear context boundary (what it knows, can do, owns)
- Never give an agent tools it doesn't need (least-privilege)
- Human gates before irreversible actions (post, delete, send)
- Observe every tool call and its outcome
