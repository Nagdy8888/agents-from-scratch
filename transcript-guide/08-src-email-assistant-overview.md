# Overview: `src/email_assistant` Folder

This document explains the **role of each file** in `src/email_assistant`. The folder implements the email assistant in stages: basic agent → human-in-the-loop → memory → Gmail.

---

## Root-level files

| File | Purpose |
|------|--------|
| **`__init__.py`** | Package marker; exposes package `version`. |
| **`configuration.py`** | Defines a `Configuration` dataclass and `from_runnable_config()` so the agent can read config from `RunnableConfig` or environment variables. Used for configurable parameters. |
| **`schemas.py`** | **Pydantic/TypedDict schemas**: `RouterSchema` (triage: reasoning + classification), `State` / `StateInput` (graph state), `EmailData`, `UserPreferences` (for memory updates). Shared by all graph implementations. |
| **`prompts.py`** | **Prompt templates**: triage system/user prompts, agent system prompts (basic, HITL, HITL+memory), default background/triage/response/calendar text, and memory-update instructions. Used when building the graph and formatting LLM inputs. |
| **`utils.py`** | **Helpers**: `format_email_markdown`, `format_gmail_markdown` (HTML→text for Gmail), `format_for_display` (for Agent Inbox), `parse_email` / `parse_gmail` to turn email dicts into (author, to, subject, thread) for prompts. |
| **`cron.py`** | **Cron graph**: small LangGraph that has one node `ingest_emails` calling `fetch_and_process_emails` from the Gmail ingest script. Used by LangGraph Platform to run **scheduled email ingestion** (e.g. every 10 minutes). State is `JobKickoff` (email, minutes_since, graph_name, url, etc.). |

---

## Main graph implementations (the “agents”)

Each file defines a **compiled LangGraph** and is referenced in `langgraph.json`.

| File | What it is | When to use it |
|------|------------|----------------|
| **`langgraph_101.py`** | Minimal demo graph: one LLM node + one tool node (`write_email`), `MessagesState`, conditional edge. No triage, no HITL. | Learning LangGraph basics; entry point for “LangGraph 101” in Studio. |
| **`email_assistant.py`** | **Full email assistant (no HITL, no memory)**. Triage router (respond/notify/ignore) → optional response agent with tools (write_email, schedule_meeting, check_calendar, Done). Uses **default (mock) tools**. | Testing the core flow; evaluation; when you don’t need approval or memory. |
| **`email_assistant_hitl.py`** | Same as above **+ human-in-the-loop**. Interrupts: (1) when triage says “notify” (user can respond or ignore), (2) before sensitive tools (write_email, schedule_meeting, Question). Uses **default (mock) tools**. | When you want to approve/edit/ignore tool calls and notifications before they run. |
| **`email_assistant_hitl_memory.py`** | Same as HITL **+ long-term memory**. Uses LangGraph **store**: triage/response/calendar preferences, updated from HITL feedback via an LLM. Reads preferences into the agent prompt. Uses **default (mock) tools**. | When you want the assistant to learn from your feedback across threads. |
| **`email_assistant_hitl_memory_gmail.py`** | Same as HITL + memory but **Gmail/Calendar tools**: real send email, check calendar, schedule meeting. Uses `get_tools(..., include_gmail=True)` and Gmail-specific prompts/utils. | Production-style assistant connected to real Gmail and Google Calendar. |

---

## Tools: `tools/`

### Shared

| File | Purpose |
|------|--------|
| **`tools/__init__.py`** | Re-exports `get_tools`, `get_tools_by_name`, and default tools (write_email, triage_email, Done, schedule_meeting, check_calendar_availability). |
| **`tools/base.py`** | **`get_tools(tool_names, include_gmail)`**: returns a list of tools (default and optionally Gmail). **`get_tools_by_name(tools)`**: returns a dict name → tool. Used by all graph files to resolve which tools the agent has. |

### Default (mock) tools: `tools/default/`

| File | Purpose |
|------|--------|
| **`tools/default/__init__.py`** | Exports default tools and prompt template constants. |
| **`tools/default/email_tools.py`** | Mock tools: `write_email`, `triage_email`, `Done`, `Question`. No real API calls. |
| **`tools/default/calendar_tools.py`** | Mock tools: `schedule_meeting`, `check_calendar_availability`. Return placeholder strings. |
| **`tools/default/prompt_templates.py`** | Text snippets describing tools for prompts: `STANDARD_TOOLS_PROMPT`, `AGENT_TOOLS_PROMPT`, `HITL_TOOLS_PROMPT`, `HITL_MEMORY_TOOLS_PROMPT`. Injected into agent system prompts. |

### Gmail tools: `tools/gmail/`

| File | Purpose |
|------|--------|
| **`tools/gmail/__init__.py`** | Package marker for Gmail tools. |
| **`tools/gmail/README.md`** | Instructions: enable Gmail/Calendar APIs, OAuth, where to put secrets, how to run ingest and deploy. |
| **`tools/gmail/gmail_tools.py`** | **Real** tools using Gmail and Calendar APIs: send email, fetch emails, check calendar, schedule meeting, mark as read. Used when `include_gmail=True`. |
| **`tools/gmail/prompt_templates.py`** | Gmail-specific tool descriptions (e.g. `GMAIL_TOOLS_PROMPT`) for the agent prompt. |
| **`tools/gmail/setup_gmail.py`** | **OAuth setup script**: uses `.secrets/secrets.json`, runs the consent flow, writes `.secrets/token.json`. Run once per machine to get a refresh token. |
| **`tools/gmail/run_ingest.py`** | **Email ingestion**: fetches emails from Gmail (filters by address, time range, read/unread), formats them, and **invokes the deployed graph** (local or hosted URL) once per email. Used for one-off ingest or by the cron graph. |
| **`tools/gmail/setup_cron.py`** | **Cron registration**: uses LangGraph SDK to create a **scheduled job** (e.g. every 10 minutes) that runs the **cron** graph with given email/URL/graph_name. That graph then runs `fetch_and_process_emails` to ingest new emails. |

---

## Evaluation: `eval/`

| File | Purpose |
|------|--------|
| **`eval/email_dataset.py`** | **Test data**: predefined email dicts (author, to, subject, email_thread) and expected outputs (classification, tool calls, response criteria). Used by pytest and LangSmith Evaluate (e.g. `examples_triage`). |
| **`eval/evaluate_triage.py`** | **Triage evaluation**: creates a LangSmith dataset from `examples_triage`, defines a target that runs `email_assistant` on each example, and an evaluator that compares `classification_decision` to reference. Uses LangSmith Evaluate API. |
| **`eval/prompts.py`** | **Evaluation prompts**: e.g. triage classification evaluation instructions, response-criteria system prompt for LLM-as-judge (used in tests like `test_response.py`). |

---

## Summary diagram

```
src/email_assistant/
├── __init__.py, configuration.py, schemas.py, prompts.py, utils.py   # Shared config, types, prompts, helpers
├── cron.py                                                           # Graph for scheduled ingest (used by Platform cron)
├── langgraph_101.py                                                  # Minimal demo graph
├── email_assistant.py                                                # Triage + agent, mock tools
├── email_assistant_hitl.py                                           # + interrupts (HITL), mock tools
├── email_assistant_hitl_memory.py                                    # + store (memory), mock tools
├── email_assistant_hitl_memory_gmail.py                              # + Gmail/Calendar tools
├── tools/
│   ├── base.py, __init__.py                                          # get_tools, get_tools_by_name
│   ├── default/                                                      # Mock email + calendar tools + prompt templates
│   └── gmail/                                                        # Real Gmail tools, ingest, cron setup, OAuth setup
└── eval/                                                             # Dataset, triage eval, eval prompts
```

You can use this as a map: **which file defines the graph**, **which file defines the tools**, and **which file is used for evaluation or deployment**.
