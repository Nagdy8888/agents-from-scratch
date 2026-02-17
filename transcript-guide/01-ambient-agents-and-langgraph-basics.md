# Ambient Agents and LangGraph Basics

## What Are Ambient Agents?

**Ambient agents** are AI systems that work in the background instead of waiting for you to chat with them. They’re different from normal “chat” agents:

| Chat agents | Ambient agents |
|-------------|----------------|
| You ask, they answer. One request at a time. | They **listen** to events (e.g. new emails) and act automatically. |
| You wait for a quick response. | Work can take a long time; they **notify you** when they need input or when something is done. |
| One turn: request → response. | Many events at once; they can **pause** and ask for clarification, then continue. |

**Example:** An email assistant that watches your inbox, triages emails, drafts replies, and only bothers you when it needs approval or wants to tell you something important.

This course builds an **ambient email agent** that can:
- Ingest incoming emails
- Draft responses and schedule meetings (via tools)
- Use **human-in-the-loop** so you approve sensitive actions
- Use **memory** so it learns from your feedback over time

---

## How Do We Build This? Tool Calling

The core idea is **tool calling**:

1. You give the LLM access to **tools** (e.g. “send email”, “check calendar”).
2. The model **reasons** about the user’s request and decides whether to call a tool.
3. When it decides to call a tool, it outputs **structured arguments** (e.g. `subject`, `content` for “send email”).
4. Your **code** runs the actual tool with those arguments (e.g. call Gmail API).
5. The **result** can be fed back to the model so it can decide the next step.

So: the LLM never “sends” the email itself—it only produces the arguments; your code executes the tool. That keeps control and safety in your system.

---

## LangChain, LangGraph, and LangSmith

Three pieces work together:

- **LangChain** – Integrations and interfaces: chat models (`init_chat_model`), tools, structured outputs. You use it to call LLMs and bind tools.
- **LangGraph** – **Orchestration**: defines the flow of your agent (nodes and edges, workflows + agents, persistence, interrupts).
- **LangSmith** – **Observability**: tracing, debugging, and evaluation of your runs.

You use LangChain for “what the model does once,” and LangGraph for “when it runs, in what order, and how we pause or persist.”

---

## LangGraph in a Nutshell

LangGraph models your application as a **graph** with:

1. **State** – A shared object (e.g. a dict or Pydantic model) that flows through the graph. Each node can read it and **update** it (e.g. add messages, store a classification).
2. **Nodes** – Python functions that take state, do work (e.g. call the LLM, run a tool), and return **updates** to the state.
3. **Edges** – Define **control flow**: which node runs next. Edges can be:
   - **Fixed:** “always go from A to B.”
   - **Conditional:** “if the last message is a tool call, go to `run_tool`, else go to `end`.”

So you **control** the flow explicitly (unlike a single black-box LLM call), and you can **persist** state and **interrupt** for human input.

---

## Chat Models and Tools in LangChain

- **Chat models** – Take a list of **messages** and return a message. LangChain’s `init_chat_model(provider, model, ...)` gives you a common interface (e.g. `invoke` with a string or messages).
- **Tools** – Functions you expose to the model. You can create them with the `@tool` decorator; the model gets descriptions and argument schemas. You **bind** tools to the model with `model.bind_tools(tools)`; you can force “must call a tool” or “one tool at a time” with parameters.
- **Execution** – When the model returns a **tool call** (name + arguments), your code runs the real tool and can append a **tool message** with the result so the model can continue.

This “model suggests tool call → code runs tool → result back to model” loop is the basis of both **workflows** (fixed steps) and **agents** (loop until done).

---

## Persistence and Checkpointing

**Persistence** means saving the **state** of the graph so you can stop and resume later. That’s what enables:

- **Human-in-the-loop** – Pause, show something to the user, wait for feedback, then resume with the same state.
- **Long-running or background runs** – Stop and continue later without losing context.

In LangGraph this is done with a **checkpointer**. After each node, the checkpointer saves the current state. When you **interrupt** (e.g. for human approval), that state is stored. When the user responds, you **resume** from that checkpoint with the same state.

You always run with a **thread_id** so all checkpoints for one conversation or task are grouped together.

---

## Interrupts

An **interrupt** is a deliberate pause in the graph. In a node you call something like `interrupt(value)` (the exact API may vary). The graph then:

1. Saves state (checkpointer).
2. Returns control to you with the **interrupt value** (e.g. “please approve this email”).
3. When you’re ready, you **resume** by invoking the graph again with a **command** that includes the user’s response (e.g. “approved” or edited content). That value is passed into the next step so the graph can continue with the right information.

So: **interrupt** = “pause and hand off to the user”; **resume** = “continue with user input.”

---

## LangGraph Studio and Local Deployment

- **LangGraph Studio** – A visual IDE to run your graph, inspect state at each node, and see traces. Very useful for debugging.
- **Local deployment** – Running `langgraph dev` in your repo starts a local server that serves your graphs (as defined in `langgraph.json`). Studio connects to this server so you can test your agent in a UI. Threads and (in simple setups) store can be stored locally on your machine.

Together, these give you a clear path: build in code → run locally with `langgraph dev` → inspect in Studio → later deploy to a hosted environment.

---

## Summary

- **Ambient agents** react to events, run in the background, and can pause for human input.
- **Tool calling** is the core: the LLM outputs structured tool arguments; your code runs the tools.
- **LangGraph** gives you state, nodes, and edges so you can combine workflows and agents and add persistence and interrupts.
- **Persistence (checkpointing)** and **interrupts** are what make human-in-the-loop and resumable runs possible.
- **LangChain** provides the LLM and tool interfaces; **LangSmith** provides tracing and evaluation; **LangGraph** orchestrates the flow.

Next: [Workflows vs Agents](./02-workflows-vs-agents.md) — when to use which and how they relate to tool calling.
