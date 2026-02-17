# Transcript Guide — Building Ambient Agents with LangGraph

This folder contains **detailed, easy-to-understand guides** based on the course transcript (`transcript.txt`). Each markdown file explains one major topic so you can learn at your own pace.

## How to Use These Guides

- Read in order **01 → 07** for a full path from basics to deployment.
- Or jump to a topic you care about; each file ends with a short summary and a link to the next.
- The tone is explanatory and concept-first, with minimal jargon. Code is referenced only where it helps clarity.

## Contents

| File | Topic |
|------|--------|
| [01-ambient-agents-and-langgraph-basics.md](01-ambient-agents-and-langgraph-basics.md) | What ambient agents are, tool calling, LangChain/LangGraph/LangSmith, nodes/edges/state, persistence, interrupts, Studio and local dev |
| [02-workflows-vs-agents.md](02-workflows-vs-agents.md) | Workflows vs agents, when to use which, control flow, and how LangGraph supports both |
| [03-building-the-email-assistant.md](03-building-the-email-assistant.md) | Building the email assistant: triage router, response agent, state, composing workflow + agent in one graph |
| [04-evaluation-and-testing.md](04-evaluation-and-testing.md) | Evaluation with LangSmith: unit tests (triage, tool calls), datasets and Evaluate API, LLM-as-judge for response quality |
| [05-human-in-the-loop.md](05-human-in-the-loop.md) | Human-in-the-loop: interrupts, request/response shape, Agent Inbox, accept/edit/ignore/feedback, keeping message history consistent |
| [06-memory.md](06-memory.md) | Memory: thread-scoped vs long-term, the store, when and how to update preferences from feedback, using an LLM to update memory |
| [07-deployment-and-gmail.md](07-deployment-and-gmail.md) | Deployment: local vs hosted, repo structure, checkpointer/store in deployment, Gmail setup, ingest, cron, Agent Inbox with hosted deployment |

## Source

These guides are derived from the **LangChain Academy** course *Building Ambient Agents with LangGraph* (transcript in `transcript.txt` at the repo root). They are meant to make the same material easier to follow in written form.
