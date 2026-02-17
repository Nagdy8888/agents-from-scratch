# Transcript Guide — Building Ambient Agents with LangGraph

This folder contains **detailed, easy-to-understand guides** based on the course transcript (`transcript.txt`) plus **reference docs** for this repo (structure, tests, troubleshooting). Use them to learn concepts and to work with the codebase day to day.

## How to Use These Guides

- **Learning path:** Read **01 → 07** in order for a full path from basics to deployment.
- **Reference:** Use **08–12** when you need to find a file, run tests, fix a problem, or start a new project.
- Each concept file (01–07) ends with a short summary and a link to the next; reference files (08–12) stand alone.

## Contents

### Concept guides (from transcript)

| File | Topic |
|------|--------|
| [01-ambient-agents-and-langgraph-basics.md](01-ambient-agents-and-langgraph-basics.md) | What ambient agents are, tool calling, LangChain/LangGraph/LangSmith, nodes/edges/state, persistence, interrupts, Studio and local dev |
| [02-workflows-vs-agents.md](02-workflows-vs-agents.md) | Workflows vs agents, when to use which, control flow, and how LangGraph supports both |
| [03-building-the-email-assistant.md](03-building-the-email-assistant.md) | Building the email assistant: triage router, response agent, state, composing workflow + agent in one graph |
| [04-evaluation-and-testing.md](04-evaluation-and-testing.md) | Evaluation with LangSmith: unit tests (triage, tool calls), datasets and Evaluate API, LLM-as-judge for response quality |
| [05-human-in-the-loop.md](05-human-in-the-loop.md) | Human-in-the-loop: interrupts, request/response shape, Agent Inbox, accept/edit/ignore/feedback, keeping message history consistent |
| [06-memory.md](06-memory.md) | Memory: thread-scoped vs long-term, the store, when and how to update preferences from feedback, using an LLM to update memory |
| [07-deployment-and-gmail.md](07-deployment-and-gmail.md) | Deployment: local vs hosted, repo structure, checkpointer/store in deployment, Gmail setup, ingest, cron, Agent Inbox with hosted deployment |

### Reference guides (this repo)

| File | Topic |
|------|--------|
| [08-src-email-assistant-overview.md](08-src-email-assistant-overview.md) | Role of each file in `src/email_assistant`: graphs, tools, eval, Gmail. |
| [09-root-level-files-explained.md](09-root-level-files-explained.md) | Root-level files explained in plain language: `__init__.py`, configuration, schemas, prompts, utils, cron. |
| [10-starting-a-new-ai-agent-project.md](10-starting-a-new-ai-agent-project.md) | Step-by-step: new project structure, pyproject.toml, uv and uv.lock, .env, optional LangGraph and Docker. |
| [11-notebooks-and-tests-guide.md](11-notebooks-and-tests-guide.md) | What each notebook does; how to run script and notebook tests; where test/eval data and evaluators live. |
| [12-troubleshooting-and-quick-reference.md](12-troubleshooting-and-quick-reference.md) | Common errors and fixes; one-page quick reference (commands, paths, env vars). |

## Source

Concept guides (01–07) are derived from the **LangChain Academy** course *Building Ambient Agents with LangGraph* (transcript in `transcript.txt` at the repo root). Reference guides (08–12) are written for this repository and for reusing the same patterns in new projects.
