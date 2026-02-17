# Workflows vs Agents

## Why It Matters

Not every task needs the same amount of flexibility. Sometimes the steps are fixed; sometimes the model should decide the steps. LangGraph lets you choose:

- **Workflows** – Predetermined steps and branching (e.g. “always triage, then maybe respond”). Lower latency and cost, easier to reason about.
- **Agents** – The model chooses which tools to call and in what order, in a loop. More flexible, better when the right sequence isn’t known in advance.

Understanding the difference helps you pick the right design and control cost and behavior.

---

## Simple System: Always Send an Email

Imagine: for every incoming email, the system **always** calls one tool — “send email” — with the LLM drafting the content. Flow:

1. Incoming email → LLM (with one tool: write/send email) → tool runs → done.

**Characteristics:**

- **Low agency** – Only one decision: what to put in the email. No “should I reply?” or “which tool?”
- **High predictability** – You always get a single tool call and then exit.

This is a **fixed workflow**: one LLM call, one tool, then stop.

---

## Adding a Router: Triage First

Next step: add a **router** that runs **before** the “send email” step. The router decides: *respond*, *notify user*, or *ignore*.

Flow:

1. Incoming email → **Router (LLM or logic)** → decision.
2. If “respond” → LLM with send-email tool → tool runs → done.
3. If “notify” or “ignore” → done (no email sent).

**Characteristics:**

- **More agency** – The system can choose not to respond.
- **Still predictable** – The path is “router → one of a few fixed branches.” You can draw this on a whiteboard.

This is still a **workflow**: predefined steps and branches, with an LLM used inside those steps.

---

## Full Agent: The Model Chooses the Sequence of Tools

Now give the LLM **several tools** (e.g. check calendar, schedule meeting, write email, done) and let it **loop**:

1. LLM sees the email (and maybe previous tool results).
2. LLM chooses a tool and outputs arguments.
3. You run that tool and append the result as a message.
4. Repeat until the LLM calls something like “done” or stops calling tools.

So the model might: check calendar → schedule meeting → write email → done. Or: write email → done. Or: check calendar → write email → done. The **sequence is not fixed**; the model decides it at runtime.

**Characteristics:**

- **High agency** – Many possible sequences of tool calls.
- **Lower predictability** – You don’t know in advance which tools will be called or how many times.

This is an **agent**: the same “call model → run tool → feed back” loop, but the **control flow** (which tool next) is determined by the model, not by your code.

---

## When to Use Which?

A simple heuristic from the course:

- **Workflow** – You can **draw the control flow** easily and list the possible paths in advance. Use when:
  - Steps and branches are known (e.g. “always triage, then maybe respond”).
  - You want **lower latency and cost** and simpler behavior.
- **Agent** – You **cannot** know at design time exactly which tools will be used or in what order. Use when:
  - The right action depends strongly on content (e.g. this email needs a meeting, that one just a short reply).
  - You’re okay with **more reasoning** (and a bit more latency/cost) for flexibility.

In practice you often **combine** them: e.g. a **workflow** that does triage (router) and then hands off to an **agent** that handles the actual response with multiple tools.

---

## How LangGraph Fits In

- **Workflows** in LangGraph: you define **nodes** (e.g. `triage`, `call_llm`, `run_tool`) and **edges** (fixed or conditional). The graph structure is fixed; nodes may call the LLM or tools inside them.
- **Agents** in LangGraph: you typically have a “call model” node and a “run tool” node, with a **conditional edge** that either loops back to the model or goes to “end” (e.g. when the model doesn’t call a tool or calls “done”).

So: **same building blocks** (state, nodes, edges, tools), but workflows have a **fixed** graph, while agents have a **loop** where the model decides the next step each time.

---

## Summary

| Concept | Meaning |
|--------|--------|
| **Workflow** | Fixed or easy-to-draw steps; LLM used inside those steps. Predictable, often cheaper. |
| **Agent** | Model chooses tool sequence in a loop. Flexible, better when the right sequence isn’t known in advance. |
| **Tool calling** | Model outputs tool name + args; your code runs the tool and can feed results back. |
| **Router** | A step that decides which branch to take (e.g. respond / notify / ignore). |

Next: [Building the Email Assistant](./03-building-the-email-assistant.md) — how the course combines a triage workflow with a response agent.
