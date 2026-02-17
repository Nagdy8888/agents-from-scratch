# Building the Email Assistant

## What We’re Building

The email assistant has two main parts:

1. **Triage (workflow)** – A fixed first step: classify each email as **respond**, **notify**, or **ignore**.
2. **Response agent (agent)** – If the decision is “respond,” an agent with tools (e.g. write email, schedule meeting, check calendar, done) that decides which tools to call and in what order.

So: **workflow for triage** + **agent for responding**. The course builds this in LangGraph and then adds evaluation, human-in-the-loop, and memory.

---

## Why This Shape?

- **Triage first** – We always want one clear decision per email: respond, notify, or ignore. That’s a single, predictable step → good as a **workflow** (router).
- **Response as agent** – What to do next depends on the email: sometimes just reply, sometimes check calendar and schedule a meeting, sometimes both. The sequence isn’t fixed → good as an **agent** with a tool loop.

LangGraph makes it easy to **compose** the two: one graph with a triage node and a “response agent” that is itself a subgraph (or a set of nodes that form an agent loop).

---

## State

The graph shares a **state** object. For this assistant, state typically includes:

- **messages** – Conversation history (including the email and any “Respond to this email” instruction). The agent subgraph reads this.
- **email_input** – The raw incoming email (e.g. for triage).
- **classification_decision** – The router output: respond / notify / ignore.

You can define state as a TypedDict, a dataclass, or a Pydantic model. LangGraph’s **MessageState** (messages as a list that gets **appended** to) is often used so that each node can add messages without overwriting the rest.

---

## Triage Node (Router)

The **triage** node:

1. Reads `email_input` (and maybe `messages`) from state.
2. Calls an LLM with **structured output** (e.g. Pydantic schema) so the model returns a fixed shape: e.g. `reasoning` + `classification` (respond | notify | ignore).
3. Writes `classification_decision` into state.
4. Uses a **command** (or conditional edge) to decide the next node:
   - If **respond** → add a message like “Respond to this email” (and maybe the email) to state, then go to **response agent**.
   - If **notify** or **ignore** → go to **end** (and optionally later, in HITL, “notify” goes to an interrupt so the user sees it).

Structured output (e.g. `with_structured_output(schema)`) makes the router’s output easy to use in code and keeps the flow deterministic after the LLM call.

---

## Response Agent

The response part is an **agent**: an LLM with access to tools, running in a loop.

- **Tools** – e.g. `write_email`, `schedule_meeting`, `check_calendar_availability`, `done`. The “done” tool is used as an explicit **termination** signal: when the model calls “done,” we leave the agent loop.
- **LLM node** – Takes current `messages` (including “Respond to this email” and the email), calls the model with tools bound, appends the model’s message (and any tool call) to state.
- **Tool node** – Takes the latest tool call(s), runs the actual tools (mock or Gmail), appends **tool messages** with results to state.
- **Conditional edge** – If the last message was a “done” tool call → go to **end**. Otherwise → go back to the **LLM node** (so the model can see the tool results and call more tools or “done”).

So the agent keeps “call model → run tools → update messages” until the model says it’s done. The triage node has already put the right content in `messages` for “respond” cases.

---

## Composing Triage and Agent in One Graph

High-level structure:

1. **Start** → **triage** node.
2. **Triage** → conditional:
   - respond → **response_agent** (subgraph or set of nodes)
   - notify / ignore → **end** (or, with HITL, to an interrupt handler first).
3. **Response agent** internally: **LLM** ↔ **tool** loop until “done,” then → **end**.

So the “response agent” is just another part of the big graph; it receives state (especially `messages`) from the triage node and updates that same state. No special magic—just nodes and edges.

---

## Tools: Mock vs Real

For learning and testing, the repo uses **mock tools** (no real Gmail or calendar). They have the same interface (e.g. `write_email(subject, content, ...)`) so the graph code doesn’t change when you later **swap** to real Gmail tools in `email_assistant_hitl_memory_gmail` and the Gmail tools module. The agent logic stays the same; only the tool implementations change.

---

## Running and Inspecting in Studio

Once the graph is defined and listed in `langgraph.json`, you can run `langgraph dev` and open **LangGraph Studio**. There you can:

- Select the “email assistant” graph.
- Send a sample email.
- Watch the path: triage → (if respond) → agent loop (LLM → tools → … → done) → end.
- Inspect state at each node and open traces in LangSmith.

This gives you a clear picture of how the workflow and agent parts work together.

---

## Summary

- **Triage** = one LLM call with structured output (respond / notify / ignore); updates state and routes to the agent or end.
- **Response agent** = loop of “LLM with tools” → “run tools” → “append messages” until the model calls “done.”
- **State** (especially `messages`) is the contract between triage and the agent: triage sets up the initial “Respond to this email” (and email) in `messages`; the agent reads and updates `messages` until it’s done.
- You **compose** workflow + agent in a single LangGraph graph by adding triage as a node and the agent as a subgraph or group of nodes with conditional edges.

Next: [Evaluation and Testing](./04-evaluation-and-testing.md) — how to test triage, tool calls, and full responses with LangSmith.
