# Root-Level Files Explained (Simple Version)

This guide explains each root-level file in `src/email_assistant` in plain language: **what it is**, **why it exists**, and **how it’s used**.

---

## 1. `__init__.py` — “This folder is a Python package”

**What it does:**  
It only defines `version = "0.1.0"`. Its main job is to tell Python: “the folder `email_assistant` is a **package**,” so you can write:

```python
from email_assistant.prompts import triage_system_prompt
from email_assistant.schemas import State
```

**Why it exists:**  
In Python, a folder with an `__init__.py` is treated as a package. Without it, you couldn’t import from `email_assistant` the way the rest of the project does. The version string is optional (e.g. for display or packaging).

**In one sentence:**  
Makes `email_assistant` a proper Python package and optionally stores its version.

---

## 2. `configuration.py` — “Where do we get settings from?”

**What it does:**  
Defines a small **Configuration** class that builds config from:

1. **Environment variables** (e.g. `MY_PARAM` in the OS)
2. **RunnableConfig** — a dict LangGraph passes when you run a graph (often contains things like `thread_id`, or custom keys you set)

So instead of hard-coding settings in code, you can say: “read this value from the environment or from the config that was passed when the graph was invoked.”

**Why it exists:**  
Right now the class is a **placeholder**: it has no real fields (no `api_key`, no `model_name`, etc.). So today it doesn’t change behavior much. It’s there so that **later** you can add things like:

- `model_name` → from env or config  
- `max_retries` → from env or config  

and the graph can use `Configuration.from_runnable_config(config)` to get those values in one place.

**In one sentence:**  
Central place to define “where config comes from” (env + LangGraph config); currently minimal, ready for you to add real settings.

---

## 3. `schemas.py` — “What shape does the data have?”

**What it does:**  
Defines the **shapes** of data the rest of the code uses. No logic, just “this object has these fields and types.”

- **RouterSchema**  
  Used when the **triage** step calls the LLM. We want the model to return exactly two things:
  - `reasoning` (string) — why it chose this category  
  - `classification` — one of `"ignore"`, `"respond"`, `"notify"`  

  So we give the LLM this schema; the LLM’s answer is forced into that shape. The code can then do `result.classification` and `result.reasoning` without guessing.

- **State**  
  The **graph state**: what every node reads and writes. It has:
  - `messages` (from LangGraph’s MessagesState) — the conversation + tool calls  
  - `email_input` — the raw email (author, to, subject, body, etc.)  
  - `classification_decision` — the triage result: `"ignore"` | `"respond"` | `"notify"`  

  So every node knows: “state has at least these keys and types.”

- **StateInput**  
  TypedDict for the **initial input** when you invoke the graph. Right now it just says “input has `email_input` (a dict).” So when you call `graph.invoke({"email_input": {...}})`, the type checker and the graph know what to expect.

- **EmailData**  
  Another shape for email data (id, thread_id, from_email, subject, etc.). Used when the data comes from somewhere (e.g. Gmail) and you want a clear contract: “an email looks like this.”

- **UserPreferences**  
  Used when we **update memory** from human feedback. The LLM that updates preferences is asked to return:
  - `chain_of_thought` — short reasoning  
  - `user_preferences` — the new preference text (string)  

  So we get a consistent structure instead of free-form text.

**Why it exists:**  
So that:
- The triage LLM returns a fixed shape (RouterSchema).  
- The graph state and inputs are well-defined (State, StateInput).  
- Email and preference updates have a clear shape (EmailData, UserPreferences).  

That avoids “magic dicts” and makes it easier to change one place when the shape changes.

**In one sentence:**  
Defines the “contracts” for triage output, graph state, email data, and memory updates so the rest of the code can rely on known fields and types.

---

## 4. `prompts.py` — “What do we tell the LLM?”

**What it does:**  
Holds all **text** we send to the LLM: system prompts, user prompts, and default text we plug into them. No logic—just strings (often with placeholders like `{background}`, `{tools_prompt}`).

- **Triage prompts**  
  - `triage_system_prompt` — “You triage emails. Categories: IGNORE, NOTIFY, RESPOND. Use background and rules below.”  
  - `triage_user_prompt` — “From: {author}, To: {to}, Subject: {subject}, body: {email_thread}.”  

  The graph fills in `{author}`, `{to}`, etc. from the email and passes the result to the triage LLM.

- **Agent prompts**  
  - `agent_system_prompt` — for the basic assistant (no HITL, no memory).  
  - `agent_system_prompt_hitl` — adds instructions like “use the Question tool when you need to ask the user.”  
  - `agent_system_prompt_hitl_memory` — same idea, used when we also have memory (response/calendar/triage preferences).  

  Each contains placeholders like `{tools_prompt}`, `{background}`, `{response_preferences}`, `{cal_preferences}`. When we build the graph, we fill those from tool descriptions and from memory (or defaults).

- **Defaults**  
  - `default_background`, `default_triage_instructions`, `default_response_preferences`, `default_cal_preferences` — used when we don’t have memory yet or when we’re not using memory.  
  - `MEMORY_UPDATE_INSTRUCTIONS` (+ reinforcement) — instructions for the **LLM that updates memory**: “don’t overwrite the whole profile, only add/update what the feedback says, preserve the rest.”

**Why it exists:**  
So we don’t scatter long prompt strings inside the graph code. We change behavior (triage rules, assistant personality, memory rules) by editing prompts in one file. The graph code just does “format this prompt with these variables and send to the LLM.”

**In one sentence:**  
Single place for all prompts and default texts; the graph fills placeholders and sends these strings to the LLMs.

---

## 5. `utils.py` — “Small helpers so the graph doesn’t get messy”

**What it does:**  
Provides small functions that other files use so the graph and tools stay readable.

- **Formatting for humans**
  - `format_email_markdown(subject, author, to, email_thread, ...)` — turns email pieces into one markdown string (Subject, From, To, body). Used when we show the email in the graph or in Agent Inbox.  
  - `format_gmail_markdown(...)` — same idea but if the body is **HTML**, it converts HTML → readable text first (so the LLM and the UI don’t see raw `<p>` tags).  
  - `format_for_display(tool_call)` — given a tool call (e.g. `write_email` or `schedule_meeting`), returns a short, readable summary (e.g. “Email Draft — To: …, Subject: …, content…”). Used when we show “this is what the agent wants to do” in Agent Inbox.

- **Parsing email input**
  - `parse_email(email_input)` — takes the dict we use as “one email” (author, to, subject, email_thread) and returns a tuple `(author, to, subject, email_thread)` so the prompt formatter can do `triage_user_prompt.format(author=..., to=..., ...)`.  
  - `parse_gmail(email_input)` — same idea for the **Gmail** shape (e.g. from, to, subject, body, id). Returns five values including `email_id` for Gmail-specific logic.

- **Messages and tool calls**
  - `extract_message_content(message)` — gets a clean string from a message (handles both plain text and the list-of-blocks format some LLMs use).  
  - `extract_tool_calls(messages)` — from a list of messages, collects all tool call **names** (e.g. `["write_email", "done"]`). Used in evaluation (“did the agent call the right tools?”).  
  - `format_messages_string(messages)` — turns a list of messages into one string (e.g. for logging or for an LLM-as-judge that “reads the conversation”).

- **Evaluation / notebooks**
  - `format_few_shot_examples(examples)` — formats examples for few-shot prompts.  
  - `show_graph(graph)` — draws the LangGraph (e.g. in a Jupyter notebook) so you can see the nodes and edges.

**Why it exists:**  
So the graph nodes and tools don’t repeat the same formatting/parsing logic. One place to fix “how we show an email” or “how we parse Gmail input” and everyone uses it.

**In one sentence:**  
Helper functions for formatting emails, parsing input, and inspecting messages/tool calls so the main graph and tools stay simple.

---

## 6. `cron.py` — “The graph that runs email ingest on a schedule”

**What it does:**  
Defines a **very small** LangGraph with a single node. That node runs the **email ingestion** process: “fetch recent emails from Gmail and, for each one, call the main email-assistant graph.”

- **JobKickoff**  
  The **state** for this small graph. It’s basically the “arguments” for the job:
  - `email` — which Gmail address to fetch from  
  - `minutes_since` — how far back to look (e.g. 60 = last hour)  
  - `graph_name` — which graph to invoke per email (e.g. `"email_assistant_hitl_memory_gmail"`)  
  - `url` — where the LangGraph server is (e.g. `http://127.0.0.1:2024` or your deployed URL)  
  - Plus options like `include_read`, `rerun`, etc.

- **main(state)**  
  The only node. It takes `JobKickoff`, builds an “args” object that the ingest script expects, and calls `fetch_and_process_emails(args)`. That function (in `tools/gmail/run_ingest.py`) talks to Gmail, gets emails, and for each email calls your deployed graph at `url`. So “cron graph” = “run the ingest script with these parameters.”

- **Graph build**  
  The file then builds: `StateGraph(JobKickoff)` → one node `ingest_emails` (the `main` function) → compile. So when the **platform** runs the “cron” graph on a schedule (e.g. every 10 minutes), it passes in a `JobKickoff` (email, url, graph_name, …) and the graph runs the ingest once.

**Why it exists:**  
So you don’t have to manually run “fetch my last 60 minutes of email and process each” every time. The deployment platform can schedule this graph (cron); each run = one ingest job with fixed parameters.

**In one sentence:**  
Defines the scheduled “job”: one node that runs the Gmail ingest script with parameters (email, time range, which graph, which URL), so the platform can run it every N minutes.

---

## How they work together (short)

- **`__init__.py`** — lets the rest of the project `import` from `email_assistant`.  
- **`configuration.py`** — (future) central place for config from env/config.  
- **`schemas.py`** — defines shapes for triage output, state, email data, and memory updates.  
- **`prompts.py`** — defines what we tell each LLM; graph fills placeholders and uses these strings.  
- **`utils.py`** — formatting and parsing so the graph and tools don’t repeat the same code.  
- **`cron.py`** — the tiny graph that “runs email ingest on a schedule” when the platform runs the cron job.

If you open any **graph file** (e.g. `email_assistant.py` or `email_assistant_hitl_memory.py`), you’ll see it **import** from these: `State`, `RouterSchema` from schemas; `triage_system_prompt`, `agent_system_prompt_...`, `default_...` from prompts; `parse_email`, `format_email_markdown`, `format_for_display` from utils. So the root-level files are the **shared building blocks**; the graph files are the **orchestration** that use them.
