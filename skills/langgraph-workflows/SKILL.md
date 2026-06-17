---
name: langgraph-workflows
description: Expert LangGraph skill for stateful multi-agent workflows — invoke when building graph-based agents, human-in-the-loop flows, checkpointed resumable pipelines, supervisor/swarm multi-agent systems, or any LLM workflow that requires cycles or conditional routing.
---

# LangGraph Workflows — Expert Reference

## Core Concepts

```
┌────────────────────────────────────────────────────────────────┐
│                       LangGraph Model                          │
│                                                                │
│   State (TypedDict / Pydantic)                                 │
│   ┌──────────────────────────────────────────────────────┐     │
│   │  messages: list[BaseMessage]   (append-only)         │     │
│   │  context:  str                 (overwrite)           │     │
│   │  step:     int                 (overwrite)           │     │
│   └──────────────────────────────────────────────────────┘     │
│                           │                                    │
│          ┌────────────────▼────────────────┐                   │
│          │          StateGraph              │                   │
│          │                                 │                   │
│          │   START ──▶ [node_a] ──▶ [node_b] ──▶ END           │
│          │                │                                    │
│          │                └──▶ [node_c] ──▶ [node_b]           │
│          │                     (conditional edge)              │
│          └─────────────────────────────────┘                   │
│                                                                │
│   Checkpointer (optional persistence layer)                    │
│   ┌──────────────────────────────────────────────────────┐     │
│   │  MemorySaver / SqliteSaver / PostgresSaver           │     │
│   │  thread_id → [checkpoint_0, checkpoint_1, ...]       │     │
│   └──────────────────────────────────────────────────────┘     │
└────────────────────────────────────────────────────────────────┘
```

## Why LangGraph vs Plain Chains

```
Chains (LCEL):                     LangGraph:
  A → B → C → D                      A → B → C
  Linear, no cycles                      ↑   │
  No native persistence                  └───┘ (cycles OK)
  No human-in-the-loop                 Conditional routing
  One-shot execution                   Checkpointing / resume
                                       Human-in-the-loop pause
                                       Time-travel debugging
                                       Parallel fan-out
```

Use LangGraph when:
- Agent must decide whether to loop (call more tools, refine output)
- You need to pause and ask a human for approval mid-execution
- Long-running workflows that must survive restarts
- Multi-agent coordination (supervisor, handoffs, swarms)
- Debugging requires replaying or forking execution history

---

## State Definition

```python
from typing import Annotated, TypedDict
import operator
from langchain_core.messages import BaseMessage

# TypedDict state (simpler, recommended for most cases)
class AgentState(TypedDict):
    # Annotated with operator.add → new values are APPENDED to existing list
    messages: Annotated[list[BaseMessage], operator.add]
    # No annotation → new value OVERWRITES existing
    context: str
    iteration_count: int

# Pydantic state (use when you need validation)
from pydantic import BaseModel, Field

class WorkflowState(BaseModel):
    messages: Annotated[list[BaseMessage], operator.add] = Field(default_factory=list)
    final_answer: str = ""
    confidence: float = 0.0
```

### Reducer Semantics

```
Annotated[list, operator.add]:
  State:  messages = [msg1, msg2]
  Update: {"messages": [msg3]}
  Result: messages = [msg1, msg2, msg3]   ← appended

No annotation (default overwrite):
  State:  context = "old context"
  Update: {"context": "new context"}
  Result: context = "new context"          ← replaced
```

---

## Building a Graph

```python
from langgraph.graph import StateGraph, START, END
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage, SystemMessage

model = ChatOpenAI(model="gpt-4o", temperature=0)

# 1. Define node functions (receive full state, return partial update dict)
def call_model(state: AgentState) -> dict:
    response = model.invoke(state["messages"])
    return {"messages": [response]}  # appended to messages list

def should_continue(state: AgentState) -> str:
    """Routing function: return name of next node."""
    last_message = state["messages"][-1]
    if last_message.tool_calls:
        return "tools"
    return END  # signal graph completion

def call_tools(state: AgentState) -> dict:
    tool_results = []
    for tool_call in state["messages"][-1].tool_calls:
        tool = tool_map[tool_call["name"]]
        result = tool.invoke(tool_call["args"])
        tool_results.append(ToolMessage(
            content=str(result),
            tool_call_id=tool_call["id"],
        ))
    return {"messages": tool_results}

# 2. Build the graph
builder = StateGraph(AgentState)

# Add nodes
builder.add_node("agent", call_model)
builder.add_node("tools", call_tools)

# Add edges
builder.add_edge(START, "agent")           # entry point

builder.add_conditional_edges(             # conditional routing
    "agent",                               # from node
    should_continue,                       # routing function
    {                                      # map: return value → next node
        "tools": "tools",
        END: END,
    },
)

builder.add_edge("tools", "agent")         # loop back to agent after tools

# 3. Compile
graph = builder.compile()

# 4. Run
result = graph.invoke({
    "messages": [HumanMessage(content="What is the weather in Paris?")],
    "context": "",
    "iteration_count": 0,
})
```

---

## ReAct Agent Pattern in LangGraph

```
┌─────────┐       ┌──────────────┐
│  START  │──────▶│    agent     │
└─────────┘       └──────┬───────┘
                         │
              ┌──────────▼──────────┐
              │  tool_calls in      │
              │  last message?      │
              └──────┬──────────────┘
                 YES │          NO
                     │          │
              ┌──────▼──────┐   ▼
              │    tools    │  END
              └──────┬──────┘
                     │
                     └──────────▶ agent (loop)
```

```python
from langgraph.prebuilt import create_react_agent

# Shortcut: prebuilt ReAct agent
graph = create_react_agent(
    model=ChatOpenAI(model="gpt-4o"),
    tools=[get_weather, web_search],
    state_modifier="You are a helpful assistant.",  # system prompt
)

result = graph.invoke({
    "messages": [HumanMessage(content="Weather in Paris and London?")]
})
```

---

## Human-in-the-Loop

```
Graph pauses before a node, yields control to human:

  agent ──▶ [PAUSE] ──▶ sensitive_tool ──▶ agent ──▶ END
                │
                ▼
           Human reviews
           Approves / Rejects / Edits
                │
                ▼
           graph.update_state() → resume
```

```python
from langgraph.checkpoint.memory import MemorySaver

checkpointer = MemorySaver()

graph = builder.compile(
    checkpointer=checkpointer,
    interrupt_before=["sensitive_tool"],  # pause BEFORE this node
    # interrupt_after=["agent"],          # pause AFTER this node
)

config = {"configurable": {"thread_id": "session-42"}}

# Run until interruption
events = graph.invoke(
    {"messages": [HumanMessage(content="Delete all old records")]},
    config=config,
)
# Graph paused — sensitive_tool not yet called

# Inspect current state
state = graph.get_state(config)
print(state.next)           # ("sensitive_tool",)
print(state.values)         # full state dict at pause point

# Option 1: approve — resume with None (no state change)
result = graph.invoke(None, config=config)

# Option 2: modify — inject human decision into state before resuming
graph.update_state(
    config,
    values={"messages": [HumanMessage(content="Actually, only delete records older than 2 years")]},
    as_node="agent",  # treat this update as if it came from "agent"
)
result = graph.invoke(None, config=config)

# Option 3: reject — go to END by updating state appropriately
graph.update_state(config, values={"messages": [AIMessage(content="Operation cancelled by user.")]})
```

---

## Checkpointing & Persistence

```python
from langgraph.checkpoint.memory import MemorySaver
from langgraph.checkpoint.sqlite import SqliteSaver
from langgraph.checkpoint.postgres import PostgresSaver

# In-memory (development / testing only — lost on restart)
checkpointer = MemorySaver()

# SQLite (single-server persistence)
with SqliteSaver.from_conn_string("checkpoints.db") as checkpointer:
    graph = builder.compile(checkpointer=checkpointer)

# PostgreSQL (production — supports concurrent workers)
with PostgresSaver.from_conn_string("postgresql://user:pass@host/db") as checkpointer:
    graph = builder.compile(checkpointer=checkpointer)

# thread_id identifies a conversation / session
config = {"configurable": {"thread_id": "user-123-session-7"}}

# Each invoke() saves a checkpoint
graph.invoke({"messages": [HumanMessage("Hello")]}, config=config)
graph.invoke({"messages": [HumanMessage("Follow-up question")]}, config=config)

# Retrieve full state history
history = list(graph.get_state_history(config))
# history[0] is most recent, history[-1] is oldest
```

---

## Time-Travel Debugging

```python
# Get all checkpoints for a thread
history = list(graph.get_state_history(config))

for snapshot in history:
    print(f"Step: {snapshot.metadata.get('step')}")
    print(f"Checkpoint ID: {snapshot.config['configurable']['checkpoint_id']}")
    print(f"Next nodes: {snapshot.next}")

# Replay from a specific checkpoint
old_checkpoint_config = history[3].config  # e.g., 4 steps ago

# Option 1: view state at that point
old_state = graph.get_state(old_checkpoint_config)

# Option 2: fork — continue from that checkpoint (creates new branch)
fork_config = {
    "configurable": {
        "thread_id": "user-123-session-7-fork",
        "checkpoint_id": old_checkpoint_config["configurable"]["checkpoint_id"],
    }
}
result = graph.invoke(
    {"messages": [HumanMessage("Try a different approach")]},
    config=fork_config,
)
```

---

## Streaming

```python
config = {"configurable": {"thread_id": "session-1"}}

# stream_mode="values" — emit full state after each node
for state in graph.stream(
    {"messages": [HumanMessage("What's the weather?")]},
    config=config,
    stream_mode="values",
):
    state["messages"][-1].pretty_print()

# stream_mode="updates" — emit only the state delta from each node
for node_name, update in graph.stream(
    {"messages": [HumanMessage("What's the weather?")]},
    config=config,
    stream_mode="updates",
):
    print(f"[{node_name}] updated: {update}")

# stream_mode="messages" — emit individual LLM tokens (for real-time UI)
async for event in graph.astream_events(
    {"messages": [HumanMessage("What's the weather?")]},
    config=config,
    version="v2",
):
    if event["event"] == "on_chat_model_stream":
        chunk = event["data"]["chunk"]
        print(chunk.content, end="", flush=True)
```

---

## Multi-Agent Patterns

### 1. Supervisor Pattern

```
┌─────────────────────────────────────────────────────┐
│                  Supervisor Agent                    │
│  (decides which specialist to call next)             │
└─────────────────┬───────────────────────────────────┘
                  │
       ┌──────────┼──────────┐
       │          │          │
┌──────▼──┐ ┌────▼────┐ ┌──▼──────┐
│Researcher│ │  Coder  │ │ Writer  │
│ subgraph │ │subgraph │ │subgraph │
└──────────┘ └─────────┘ └─────────┘
```

```python
from langgraph.graph import StateGraph, START, END

class SupervisorState(TypedDict):
    messages: Annotated[list[BaseMessage], operator.add]
    next_agent: str

def supervisor(state: SupervisorState) -> dict:
    """Decides which specialist agent to route to, or FINISH."""
    system = """You are a supervisor managing: researcher, coder, writer.
    Given the conversation, decide who acts next, or output FINISH.
    Output exactly one word: researcher, coder, writer, or FINISH."""
    response = model.invoke([SystemMessage(content=system)] + state["messages"])
    return {"next_agent": response.content.strip().lower()}

def route_supervisor(state: SupervisorState) -> str:
    return state["next_agent"] if state["next_agent"] != "finish" else END

builder = StateGraph(SupervisorState)
builder.add_node("supervisor", supervisor)
builder.add_node("researcher", researcher_agent)
builder.add_node("coder", coder_agent)
builder.add_node("writer", writer_agent)

builder.add_edge(START, "supervisor")
builder.add_conditional_edges("supervisor", route_supervisor)

# All specialists return to supervisor
for agent in ["researcher", "coder", "writer"]:
    builder.add_edge(agent, "supervisor")

graph = builder.compile()
```

### 2. Agent Handoff with Command

```python
from langgraph.types import Command

def billing_agent(state: AgentState) -> Command:
    """Billing agent that hands off to tech support if needed."""
    response = model.invoke(state["messages"])
    
    if "technical issue" in response.content.lower():
        # Hand off control to tech_support agent
        return Command(
            goto="tech_support",
            update={"messages": [response]},
        )
    return Command(
        goto=END,
        update={"messages": [response]},
    )
```

### 3. Swarm Pattern (Dynamic Handoffs)

```
Agents hand off to each other based on task content.
No central supervisor — any agent can transfer to any other.

[agent_a] ──transfer──▶ [agent_b]
    ▲                        │
    └────────transfer─────────┘
```

```python
from langgraph.prebuilt import create_react_agent
from langgraph_supervisor import create_swarm  # langgraph-supervisor package

# Each agent has a transfer tool to hand off
def transfer_to_billing(reason: str) -> str:
    """Transfer the user to the billing department."""
    return "Transferring to billing..."

billing_agent = create_react_agent(
    model,
    tools=[transfer_to_tech_support, handle_billing_query],
    name="billing_agent",
)
tech_agent = create_react_agent(
    model,
    tools=[transfer_to_billing, handle_tech_issue],
    name="tech_agent",
)
```

---

## Subgraphs

```python
# Define a subgraph with its own state
class ResearchState(TypedDict):
    query: str
    results: list[str]

research_builder = StateGraph(ResearchState)
research_builder.add_node("search", search_node)
research_builder.add_node("summarize", summarize_node)
research_builder.add_edge(START, "search")
research_builder.add_edge("search", "summarize")
research_builder.add_edge("summarize", END)

research_subgraph = research_builder.compile()

# Use as a node in the parent graph
# Parent state must overlap with subgraph state on shared keys
parent_builder.add_node("research", research_subgraph)
```

---

## Parallel Execution with Send

```python
from langgraph.types import Send

def fan_out(state: AgentState) -> list[Send]:
    """Generate a Send for each item to process in parallel."""
    topics = state["topics"]  # e.g. ["AI", "Blockchain", "Quantum"]
    return [
        Send("research_topic", {"topic": topic, "messages": []})
        for topic in topics
    ]

builder.add_conditional_edges(
    "plan",
    fan_out,
    ["research_topic"],  # declare possible destinations
)
```

---

## LangGraph Platform

```
                    LangGraph Platform
┌──────────────────────────────────────────────────────────┐
│                                                          │
│  ┌──────────────┐   ┌──────────────┐   ┌────────────┐  │
│  │  LangGraph   │   │  Deployment  │   │  Studio    │  │
│  │  Server API  │   │  (Docker /   │   │  (visual   │  │
│  │  (REST +     │   │   Cloud Run) │   │  debugger) │  │
│  │  WebSocket)  │   │              │   │            │  │
│  └──────────────┘   └──────────────┘   └────────────┘  │
│                                                          │
│  Persistent storage: PostgreSQL checkpoints              │
│  Streaming: SSE for real-time token streaming            │
│  Auth: API key or OAuth                                  │
└──────────────────────────────────────────────────────────┘
```

```bash
# langgraph.json — deployment manifest
{
  "dependencies": ["."],
  "graphs": {
    "agent": "./my_agent/graph.py:graph"
  },
  "env": ".env"
}

# Deploy locally
langgraph up

# Deploy to LangGraph Cloud
langgraph deploy
```

---

## LangGraph vs Airflow/Prefect

| Dimension          | LangGraph                         | Airflow / Prefect                  |
|--------------------|-----------------------------------|------------------------------------|
| Control flow       | LLM decides next step dynamically | Developer defines DAG statically   |
| Cycles             | Native support (agents loop)      | DAGs are acyclic by definition     |
| Trigger            | User input / event                | Schedule or external trigger       |
| State              | Rich shared state (messages etc.) | Task outputs passed explicitly     |
| Use case           | LLM-driven reasoning pipelines    | Data engineering, ETL, batch jobs  |
| Human-in-the-loop  | First-class feature               | Requires external coordination     |

---

## Anti-Patterns

### 1. Too Much State — Bloats Context Window

```python
# BAD: accumulating raw documents in state
class BadState(TypedDict):
    messages: Annotated[list[BaseMessage], operator.add]
    all_retrieved_docs: Annotated[list[Document], operator.add]  # grows forever

# GOOD: store summaries or metadata, retrieve on demand
class GoodState(TypedDict):
    messages: Annotated[list[BaseMessage], operator.add]
    doc_ids: list[str]  # reference IDs, retrieve content only when needed
```

### 2. Cycles Without Exit Condition

```python
# BAD: agent can loop forever with no iteration limit
builder.add_conditional_edges("agent", lambda s: "tools" if s["messages"][-1].tool_calls else END)

# GOOD: add iteration guard
def route_with_limit(state: AgentState) -> str:
    if state["iteration_count"] >= 10:
        return END
    if state["messages"][-1].tool_calls:
        return "tools"
    return END

def increment_counter(state: AgentState) -> dict:
    return {"iteration_count": state["iteration_count"] + 1}

builder.add_node("guard", increment_counter)
builder.add_edge("tools", "guard")
builder.add_edge("guard", "agent")
```

### 3. Synchronous Blocking in Async Graph

```python
# BAD: blocking call inside async node
async def fetch_data(state: AgentState) -> dict:
    import requests
    data = requests.get("https://api.example.com/data").json()  # blocks event loop
    return {"context": str(data)}

# GOOD: use async HTTP client
import httpx

async def fetch_data(state: AgentState) -> dict:
    async with httpx.AsyncClient() as client:
        resp = await client.get("https://api.example.com/data")
        return {"context": str(resp.json())}
```

### 4. Not Using thread_id for Multi-User Apps

```python
# BAD: all users share the same state
result = graph.invoke(input, config={"configurable": {"thread_id": "global"}})

# GOOD: unique thread per session/user
import uuid
config = {"configurable": {"thread_id": f"user-{user_id}-{session_id}"}}
result = graph.invoke(input, config=config)
```

### 5. Rebuilding Graph on Every Request

```python
# BAD: compile() on every HTTP request (expensive)
@app.post("/chat")
async def chat(request: Request):
    graph = builder.compile(checkpointer=MemorySaver())  # slow!
    return graph.invoke(...)

# GOOD: compile once at startup
checkpointer = SqliteSaver.from_conn_string("app.db")
graph = builder.compile(checkpointer=checkpointer)

@app.post("/chat")
async def chat(request: Request):
    return await graph.ainvoke(...)
```

---

## Production Checklist

- [ ] Use `SqliteSaver` or `PostgresSaver` in production (not `MemorySaver`)
- [ ] Assign unique `thread_id` per user session
- [ ] Add `iteration_count` guard to all agent loops
- [ ] Use `interrupt_before` for any irreversible actions (send email, delete data, payments)
- [ ] Prefer `ainvoke` / `astream` in async frameworks (FastAPI, etc.)
- [ ] Use `stream_mode="messages"` for real-time UI token streaming
- [ ] Set `recursion_limit` on graph compile to cap runaway recursion
- [ ] Test subgraph state overlap carefully — key name collisions cause silent bugs
- [ ] Use LangSmith to trace graph execution end-to-end
- [ ] Store minimal state — large state objects slow serialization and bloat checkpoints
