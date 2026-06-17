---
name: a2a-protocol
description: Implement Google's Agent-to-Agent (A2A) protocol for agent interoperability — invoke when the user is building multi-agent systems, agent registries, agent delegation, or integrating A2A with Spring AI or Python frameworks.
---

# Agent-to-Agent (A2A) Protocol

## What A2A Is

Google's open protocol (released April 2025) for horizontal agent-to-agent communication.

```
MCP vs A2A: Complementary, Not Competing
==========================================

MCP (Model Context Protocol)          A2A (Agent-to-Agent)
-------------------------------       ---------------------------
Agent --> Tools / Resources           Agent --> Agent
JSON-RPC 2.0 (stdio or SSE)          HTTP/JSON REST
Tool invocation                       Task delegation
No standard discovery                 Agent Cards at well-known URL
Stateless server                      Task lifecycle (submitted->done)
Vertical: capability extension        Horizontal: work distribution

Typical stack:
  Orchestrator Agent
      |--- MCP servers (tools: web search, code executor, DB)
      |--- A2A --> Code Review Agent
      |--- A2A --> Data Analysis Agent
      |--- A2A --> Notification Agent
```

A2A enables agents to delegate work **without exposing internal state, memory, tools, or system prompts** to the caller. Each agent is a black box that accepts tasks and returns artifacts.

---

## Core Concepts

```
Concept        | Description
---------------|--------------------------------------------------------------
Agent Card     | JSON manifest at /.well-known/agent.json
               | Describes: name, URL, skills, auth, capabilities
Task           | Unit of work; has ID, status, input message, output artifacts
Part           | Content unit in a message: TextPart, FilePart, DataPart
Artifact       | Output produced by a completed task (document, data, file)
Skill          | Named capability declared in Agent Card with I/O schema
Push           | Agent proactively sends updates (requires pushNotifications cap)
Streaming      | Agent streams partial results via SSE during task execution
```

### Task Status Lifecycle

```
             +-----------> input-required <----------+
             |                   |                   |
             |                   | (multi-turn)      |
             v                   v                   |
submitted --> working ---------> completed           |
                  |                                  |
                  +-----------> failed               |
                  |                                  |
                  +-----------> canceled             |
                                                     |
             (agent needs more info from caller)  ---+
```

---

## Agent Card

The Agent Card is the discovery contract. Serve it at `/.well-known/agent.json`.

```json
{
  "name": "Code Review Agent",
  "description": "Reviews pull requests for bugs, style violations, and security issues",
  "url": "https://code-review.agents.example.com",
  "version": "1.2.0",
  "documentationUrl": "https://docs.example.com/agents/code-review",
  "capabilities": {
    "streaming": true,
    "pushNotifications": false,
    "stateTransitionHistory": true
  },
  "authentication": {
    "schemes": ["Bearer"],
    "credentials": null
  },
  "defaultInputModes": ["text"],
  "defaultOutputModes": ["text", "data"],
  "skills": [
    {
      "id": "review-pr",
      "name": "Review Pull Request",
      "description": "Analyzes a PR diff and returns structured findings with severity ratings",
      "tags": ["code-review", "security", "quality"],
      "examples": [
        "Review this Python PR for security vulnerabilities",
        "Check this diff for style issues per PEP 8"
      ],
      "inputModes": ["text"],
      "outputModes": ["text", "data"],
      "inputSchema": {
        "type": "object",
        "properties": {
          "diff": {"type": "string", "description": "The PR diff in unified diff format"},
          "language": {"type": "string", "description": "Primary programming language"},
          "focus": {
            "type": "array",
            "items": {"type": "string", "enum": ["security", "style", "bugs", "performance"]},
            "description": "Areas to focus on; defaults to all"
          }
        },
        "required": ["diff"]
      }
    },
    {
      "id": "explain-code",
      "name": "Explain Code",
      "description": "Generates a plain-English explanation of a code snippet",
      "tags": ["documentation"],
      "inputModes": ["text"],
      "outputModes": ["text"]
    }
  ]
}
```

---

## A2A Endpoints

```
Endpoint                        | Method | Purpose
--------------------------------|--------|----------------------------------
/.well-known/agent.json         | GET    | Fetch Agent Card (discovery)
/tasks/send                     | POST   | Send task; wait for completion
/tasks/sendSubscribe            | POST   | Send task; stream SSE updates
/tasks/{id}                     | GET    | Get current task state
/tasks/{id}/cancel              | POST   | Cancel a running task
/tasks/{id}/send                | POST   | Continue multi-turn task
```

### Request / Response Shapes

```json
// POST /tasks/send
{
  "id": "task-uuid-1234",
  "message": {
    "role": "user",
    "parts": [
      {
        "type": "text",
        "text": "Review this diff for security issues: ..."
      },
      {
        "type": "data",
        "data": {"language": "python", "focus": ["security"]}
      }
    ]
  },
  "metadata": {
    "caller": "orchestrator-agent-v2",
    "priority": "normal"
  }
}

// Response (completed task)
{
  "id": "task-uuid-1234",
  "status": {
    "state": "completed",
    "timestamp": "2025-06-17T12:34:56Z"
  },
  "artifacts": [
    {
      "name": "review-findings",
      "description": "Security findings from code review",
      "parts": [
        {
          "type": "data",
          "data": {
            "findings": [
              {
                "severity": "high",
                "line": 42,
                "issue": "SQL injection via string interpolation",
                "recommendation": "Use parameterized queries"
              }
            ],
            "summary": "1 high severity finding"
          }
        }
      ]
    }
  ]
}
```

---

## Python Implementation: Full A2A Server

```python
# a2a_server.py
import asyncio
import uuid
from datetime import datetime, UTC
from typing import AsyncIterator
from fastapi import FastAPI, HTTPException, Header
from fastapi.responses import StreamingResponse
from pydantic import BaseModel
import json

app = FastAPI(title="Code Review A2A Agent")

# --- Data models ---

class Part(BaseModel):
    type: str          # "text" | "data" | "file"
    text: str | None = None
    data: dict | None = None

class Message(BaseModel):
    role: str
    parts: list[Part]

class TaskRequest(BaseModel):
    id: str
    message: Message
    metadata: dict | None = None

class TaskStatus(BaseModel):
    state: str
    timestamp: str
    message: Message | None = None

class Artifact(BaseModel):
    name: str
    description: str | None = None
    parts: list[Part]

class Task(BaseModel):
    id: str
    status: TaskStatus
    artifacts: list[Artifact] = []
    history: list[TaskStatus] = []

# In-memory store (use Redis/DB in production)
tasks: dict[str, Task] = {}


# --- Agent Card ---

AGENT_CARD = {
    "name": "Code Review Agent",
    "description": "Reviews code for bugs, security issues, and style violations",
    "url": "https://code-review.agents.example.com",
    "version": "1.0.0",
    "capabilities": {"streaming": True, "pushNotifications": False},
    "authentication": {"schemes": ["Bearer"]},
    "skills": [
        {
            "id": "review-pr",
            "name": "Review Pull Request",
            "description": "Analyzes a diff and returns structured findings",
            "inputModes": ["text"],
            "outputModes": ["text", "data"],
        }
    ],
}

@app.get("/.well-known/agent.json")
async def agent_card():
    return AGENT_CARD


# --- Authentication ---

async def verify_bearer(authorization: str = Header(...)) -> str:
    if not authorization.startswith("Bearer "):
        raise HTTPException(401, "Missing Bearer token")
    token = authorization.removeprefix("Bearer ")
    # Validate token against your auth system (JWT, OAuth introspection, etc.)
    if not await validate_token(token):
        raise HTTPException(403, "Invalid or expired token")
    return token


async def validate_token(token: str) -> bool:
    # Replace with real JWT validation
    return token == "demo-token"


# --- Task processing ---

async def process_review(task_request: TaskRequest) -> tuple[str, list[Artifact]]:
    """Core task logic. Returns (summary_text, artifacts)."""
    # Extract diff from message parts
    diff = ""
    focus = ["security", "bugs", "style"]
    for part in task_request.message.parts:
        if part.type == "text":
            diff = part.text or ""
        elif part.type == "data" and part.data:
            focus = part.data.get("focus", focus)

    # Call your LLM / analysis logic here
    # This is a stub:
    findings = [
        {
            "severity": "high",
            "line": 42,
            "issue": "Potential SQL injection",
            "recommendation": "Use parameterized queries",
        }
    ]
    summary = f"Found {len(findings)} issues in reviewed diff."

    artifacts = [
        Artifact(
            name="review-findings",
            description="Code review findings",
            parts=[
                Part(type="text", text=summary),
                Part(type="data", data={"findings": findings, "summary": summary}),
            ],
        )
    ]
    return summary, artifacts


@app.post("/tasks/send")
async def send_task(task_request: TaskRequest, authorization: str = Header(...)):
    await verify_bearer(authorization)

    task = Task(
        id=task_request.id,
        status=TaskStatus(
            state="working",
            timestamp=datetime.now(UTC).isoformat(),
        ),
    )
    tasks[task.id] = task

    try:
        summary, artifacts = await process_review(task_request)
        task.status = TaskStatus(
            state="completed",
            timestamp=datetime.now(UTC).isoformat(),
        )
        task.artifacts = artifacts
    except Exception as e:
        task.status = TaskStatus(
            state="failed",
            timestamp=datetime.now(UTC).isoformat(),
            message=Message(role="agent", parts=[Part(type="text", text=str(e))]),
        )

    return task


@app.post("/tasks/sendSubscribe")
async def send_task_subscribe(
    task_request: TaskRequest, authorization: str = Header(...)
):
    await verify_bearer(authorization)

    async def event_stream() -> AsyncIterator[str]:
        task = Task(
            id=task_request.id,
            status=TaskStatus(state="submitted", timestamp=datetime.now(UTC).isoformat()),
        )
        tasks[task.id] = task
        yield f"data: {task.model_dump_json()}\n\n"

        # Update to "working"
        task.status = TaskStatus(state="working", timestamp=datetime.now(UTC).isoformat())
        yield f"data: {task.model_dump_json()}\n\n"

        # Stream intermediate progress (optional)
        for i in range(3):
            await asyncio.sleep(1)
            progress = Task(
                id=task.id,
                status=TaskStatus(
                    state="working",
                    timestamp=datetime.now(UTC).isoformat(),
                    message=Message(
                        role="agent",
                        parts=[Part(type="text", text=f"Analyzing section {i+1}/3...")],
                    ),
                ),
            )
            yield f"data: {progress.model_dump_json()}\n\n"

        summary, artifacts = await process_review(task_request)
        task.status = TaskStatus(state="completed", timestamp=datetime.now(UTC).isoformat())
        task.artifacts = artifacts
        yield f"data: {task.model_dump_json()}\n\n"

    return StreamingResponse(event_stream(), media_type="text/event-stream")


@app.get("/tasks/{task_id}")
async def get_task(task_id: str, authorization: str = Header(...)):
    await verify_bearer(authorization)
    if task_id not in tasks:
        raise HTTPException(404, "Task not found")
    return tasks[task_id]


@app.post("/tasks/{task_id}/cancel")
async def cancel_task(task_id: str, authorization: str = Header(...)):
    await verify_bearer(authorization)
    if task_id not in tasks:
        raise HTTPException(404, "Task not found")
    task = tasks[task_id]
    if task.status.state not in ("submitted", "working", "input-required"):
        raise HTTPException(400, f"Cannot cancel task in state: {task.status.state}")
    task.status = TaskStatus(state="canceled", timestamp=datetime.now(UTC).isoformat())
    return task
```

### Python A2A Client

```python
# a2a_client.py
import uuid
import httpx
import json
from typing import AsyncIterator

class A2AClient:
    def __init__(self, agent_url: str, token: str):
        self.base = agent_url.rstrip("/")
        self.headers = {"Authorization": f"Bearer {token}"}

    async def get_agent_card(self) -> dict:
        async with httpx.AsyncClient() as client:
            resp = await client.get(
                f"{self.base}/.well-known/agent.json",
                headers=self.headers,
            )
            resp.raise_for_status()
            return resp.json()

    def _supports_skill(self, card: dict, skill_id: str) -> bool:
        return any(s["id"] == skill_id for s in card.get("skills", []))

    async def send_task(self, text: str, data: dict | None = None) -> dict:
        parts = [{"type": "text", "text": text}]
        if data:
            parts.append({"type": "data", "data": data})

        payload = {
            "id": str(uuid.uuid4()),
            "message": {"role": "user", "parts": parts},
        }
        async with httpx.AsyncClient(timeout=120) as client:
            resp = await client.post(
                f"{self.base}/tasks/send",
                json=payload,
                headers=self.headers,
            )
            resp.raise_for_status()
            return resp.json()

    async def stream_task(
        self, text: str, data: dict | None = None
    ) -> AsyncIterator[dict]:
        parts = [{"type": "text", "text": text}]
        if data:
            parts.append({"type": "data", "data": data})

        payload = {
            "id": str(uuid.uuid4()),
            "message": {"role": "user", "parts": parts},
        }
        async with httpx.AsyncClient(timeout=300) as client:
            async with client.stream(
                "POST",
                f"{self.base}/tasks/sendSubscribe",
                json=payload,
                headers=self.headers,
            ) as resp:
                resp.raise_for_status()
                async for line in resp.aiter_lines():
                    if line.startswith("data: "):
                        yield json.loads(line[6:])


# Usage
async def main():
    client = A2AClient("https://code-review.agents.example.com", "demo-token")

    # Discover capabilities
    card = await client.get_agent_card()
    print(f"Agent: {card['name']} -- Skills: {[s['id'] for s in card['skills']]}")

    # Stream a task
    async for update in client.stream_task(
        "Review this diff for security issues",
        {"diff": "- raw_query = f'SELECT * FROM users WHERE id = {user_id}'", "focus": ["security"]},
    ):
        print(f"[{update['status']['state']}]", end=" ")
        if update["status"].get("message"):
            print(update["status"]["message"]["parts"][0].get("text", ""))
        if update["status"]["state"] == "completed":
            for artifact in update.get("artifacts", []):
                for part in artifact["parts"]:
                    if part["type"] == "data":
                        print("Findings:", part["data"])
```

---

## Java / Spring Boot Implementation

### Agent Card Endpoint

```java
// AgentCardController.java
package com.example.a2a;

import org.springframework.web.bind.annotation.*;
import java.util.*;

@RestController
public class AgentCardController {

    @GetMapping("/.well-known/agent.json")
    public Map<String, Object> agentCard() {
        var skill = Map.of(
            "id", "review-pr",
            "name", "Review Pull Request",
            "description", "Analyzes a PR diff and returns structured findings",
            "inputModes", List.of("text"),
            "outputModes", List.of("text", "data")
        );

        return Map.of(
            "name", "Code Review Agent",
            "description", "Reviews code for bugs, security issues, and style violations",
            "url", "https://code-review.agents.example.com",
            "version", "1.0.0",
            "capabilities", Map.of(
                "streaming", true,
                "pushNotifications", false
            ),
            "authentication", Map.of("schemes", List.of("Bearer")),
            "skills", List.of(skill)
        );
    }
}
```

### Task Handler with Async Processing

```java
// TaskController.java
package com.example.a2a;

import org.springframework.http.*;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.servlet.mvc.method.annotation.SseEmitter;
import java.time.Instant;
import java.util.*;
import java.util.concurrent.*;

@RestController
@RequestMapping("/tasks")
public class TaskController {

    private final ConcurrentHashMap<String, TaskEntity> taskStore = new ConcurrentHashMap<>();
    private final ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();
    private final CodeReviewService reviewService;

    public TaskController(CodeReviewService reviewService) {
        this.reviewService = reviewService;
    }

    @PostMapping("/send")
    public ResponseEntity<TaskEntity> sendTask(
        @RequestBody TaskRequest request,
        @RequestHeader("Authorization") String auth
    ) {
        validateBearer(auth);

        var task = new TaskEntity(request.id(), "working", Instant.now());
        taskStore.put(task.id(), task);

        try {
            var result = reviewService.review(extractText(request));
            task = task.withState("completed").withArtifacts(List.of(result));
            taskStore.put(task.id(), task);
            return ResponseEntity.ok(task);
        } catch (Exception e) {
            task = task.withState("failed").withErrorMessage(e.getMessage());
            taskStore.put(task.id(), task);
            return ResponseEntity.status(500).body(task);
        }
    }

    @PostMapping("/sendSubscribe")
    public SseEmitter sendTaskSubscribe(
        @RequestBody TaskRequest request,
        @RequestHeader("Authorization") String auth
    ) {
        validateBearer(auth);
        SseEmitter emitter = new SseEmitter(300_000L);

        executor.submit(() -> {
            try {
                var task = new TaskEntity(request.id(), "submitted", Instant.now());
                emitter.send(task);

                task = task.withState("working");
                emitter.send(task);

                // Progress updates
                for (int i = 1; i <= 3; i++) {
                    Thread.sleep(1000);
                    emitter.send(task.withProgress("Analyzing section " + i + "/3"));
                }

                var result = reviewService.review(extractText(request));
                task = task.withState("completed").withArtifacts(List.of(result));
                emitter.send(task);
                emitter.complete();

            } catch (Exception e) {
                emitter.completeWithError(e);
            }
        });

        return emitter;
    }

    @GetMapping("/{taskId}")
    public ResponseEntity<TaskEntity> getTask(
        @PathVariable String taskId,
        @RequestHeader("Authorization") String auth
    ) {
        validateBearer(auth);
        var task = taskStore.get(taskId);
        return task != null
            ? ResponseEntity.ok(task)
            : ResponseEntity.notFound().build();
    }

    @PostMapping("/{taskId}/cancel")
    public ResponseEntity<TaskEntity> cancelTask(
        @PathVariable String taskId,
        @RequestHeader("Authorization") String auth
    ) {
        validateBearer(auth);
        var task = taskStore.get(taskId);
        if (task == null) return ResponseEntity.notFound().build();

        var cancelable = Set.of("submitted", "working", "input-required");
        if (!cancelable.contains(task.state())) {
            return ResponseEntity.badRequest().build();
        }
        task = task.withState("canceled");
        taskStore.put(taskId, task);
        return ResponseEntity.ok(task);
    }

    private void validateBearer(String auth) {
        if (auth == null || !auth.startsWith("Bearer ")) {
            throw new ResponseStatusException(HttpStatus.UNAUTHORIZED);
        }
        // Validate token here
    }

    private String extractText(TaskRequest request) {
        return request.message().parts().stream()
            .filter(p -> "text".equals(p.type()))
            .map(Part::text)
            .findFirst()
            .orElseThrow(() -> new IllegalArgumentException("No text part in message"));
    }
}
```

### Spring AI Integration

```java
// A2AOrchestrator.java -- Orchestrator delegating to A2A agents
package com.example.a2a;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestClient;
import java.util.*;

@Service
public class A2AOrchestrator {

    private final RestClient restClient;
    private final AgentRegistry registry;

    public A2AOrchestrator(AgentRegistry registry) {
        this.registry = registry;
        this.restClient = RestClient.create();
    }

    public Map<String, Object> delegateTask(String skillId, String taskText) {
        // 1. Discover agent by skill
        var agentUrl = registry.findBySkill(skillId)
            .orElseThrow(() -> new NoSuchElementException("No agent for skill: " + skillId));

        // 2. Verify agent card declares this skill
        var card = restClient.get()
            .uri(agentUrl + "/.well-known/agent.json")
            .retrieve()
            .body(Map.class);

        boolean hasSkill = ((List<Map<String, ?>>) card.get("skills")).stream()
            .anyMatch(s -> skillId.equals(s.get("id")));
        if (!hasSkill) throw new IllegalStateException("Agent card does not declare skill: " + skillId);

        // 3. Send task
        var payload = Map.of(
            "id", UUID.randomUUID().toString(),
            "message", Map.of(
                "role", "user",
                "parts", List.of(Map.of("type", "text", "text", taskText))
            )
        );
        return restClient.post()
            .uri(agentUrl + "/tasks/send")
            .header("Authorization", "Bearer " + getToken())
            .body(payload)
            .retrieve()
            .body(Map.class);
    }

    private String getToken() {
        // Load from secrets manager / OAuth client credentials flow
        return System.getenv("A2A_BEARER_TOKEN");
    }
}
```

---

## Agent Registry Pattern

```python
# registry.py
from dataclasses import dataclass, field
from datetime import datetime, UTC
import asyncio
import httpx

@dataclass
class RegisteredAgent:
    url: str
    card: dict
    last_seen: datetime = field(default_factory=lambda: datetime.now(UTC))
    healthy: bool = True

class AgentRegistry:
    """
    In-memory registry. In production, back with Redis or a database.
    Agents self-register or are registered by an admin.
    """
    def __init__(self):
        self._agents: dict[str, RegisteredAgent] = {}  # url -> agent

    async def register(self, agent_url: str) -> RegisteredAgent:
        card = await self._fetch_card(agent_url)
        entry = RegisteredAgent(url=agent_url, card=card)
        self._agents[agent_url] = entry
        return entry

    def find_by_skill(self, skill_id: str) -> list[RegisteredAgent]:
        return [
            agent for agent in self._agents.values()
            if agent.healthy
            and any(s["id"] == skill_id for s in agent.card.get("skills", []))
        ]

    async def health_check_all(self):
        """Run periodically (e.g. every 60s) to verify agent liveness."""
        for url, agent in self._agents.items():
            try:
                card = await self._fetch_card(url)
                agent.card = card
                agent.last_seen = datetime.now(UTC)
                agent.healthy = True
            except Exception:
                agent.healthy = False

    async def _fetch_card(self, url: str) -> dict:
        async with httpx.AsyncClient(timeout=10) as client:
            resp = await client.get(f"{url.rstrip('/')}/.well-known/agent.json")
            resp.raise_for_status()
            return resp.json()
```

---

## Multi-Turn Tasks (input-required)

```python
# When your agent needs more information from the caller

async def handle_ambiguous_task(task_request: TaskRequest) -> Task:
    """
    Return input-required status with a clarifying question.
    Caller continues via POST /tasks/{id}/send.
    """
    if needs_clarification(task_request):
        return Task(
            id=task_request.id,
            status=TaskStatus(
                state="input-required",
                timestamp=datetime.now(UTC).isoformat(),
                message=Message(
                    role="agent",
                    parts=[Part(
                        type="text",
                        text="Which programming language should I focus on? "
                             "Please reply with: python, java, go, or typescript",
                    )],
                ),
            ),
        )
    # ... process normally
```

---

## Security Checklist for A2A Agents

```
Authentication and Authorization:
[ ] All endpoints require Bearer token (OAuth 2.1 recommended)
[ ] Token validated on every request (not cached without expiry)
[ ] Caller identity extracted from token and logged
[ ] Scope/permissions checked: not every caller gets every skill

Input validation:
[ ] Task input validated against skill's inputSchema before processing
[ ] Task ID is a valid UUID; reject requests with malformed IDs
[ ] Input size limits enforced (max message length, part count)
[ ] Parts with unexpected types rejected

Output safety:
[ ] Artifacts do not contain internal state, config, or secrets
[ ] Agent Card does not expose internal infrastructure details
[ ] Error messages do not leak stack traces to caller

Task lifecycle:
[ ] Task IDs are globally unique; collision check before accepting
[ ] Concurrent task limits per caller to prevent abuse
[ ] Long-running tasks have timeouts and automatic cancellation
[ ] All state transitions logged with timestamp and caller ID

Operational:
[ ] Rate limiting per caller token
[ ] Audit log: every task received, state change, artifact delivered
[ ] Health check: /.well-known/agent.json should return < 200ms
[ ] Graceful shutdown: drain in-flight tasks before stopping
```

---

## A2A vs MCP Reference

| Dimension             | MCP                      | A2A                            |
|-----------------------|--------------------------|--------------------------------|
| Direction             | Agent to Tools/Resources | Agent to Agent                 |
| Protocol              | JSON-RPC 2.0             | HTTP/JSON REST                 |
| Transport             | stdio or SSE             | HTTP (SSE for streaming)       |
| Discovery             | No standard              | Agent Cards at well-known URL  |
| State management      | Stateless server         | Task lifecycle (submitted-done)|
| Content               | Tool calls, resources    | Tasks, artifacts, multi-turn   |
| Use case              | Capability extension     | Work delegation                |
| Spec owner            | Anthropic                | Google                         |
| Release               | November 2024            | April 2025                     |

---

## Anti-Patterns

```
[ ] Exposing internal tool names or memory in task artifacts
    --> Artifacts are external API; treat them like public endpoints

[ ] Hardcoding agent URLs instead of using registry + Agent Card discovery
    --> URLs change; Agent Cards let you find agents by capability

[ ] Sending secrets or PII in task messages
    --> Task messages are logged; use references (IDs) not raw data

[ ] Not validating skill capability before sending a task
    --> Always fetch and check Agent Card; agent may not support the skill

[ ] Skipping authentication because "agents are internal"
    --> A2A is designed for multi-tenant, cross-org agent networks; always auth

[ ] Using task ID generated by server instead of client
    --> A2A spec: client generates task ID; enables idempotency

[ ] Polling /tasks/{id} in a tight loop instead of streaming
    --> Use sendSubscribe for SSE; poll with backoff if SSE unavailable

[ ] Not handling input-required state in client
    --> Multi-turn tasks are a first-class feature; implement the continue flow
```
