# Deployment and Gmail

## Local vs Hosted Deployment

**Local deployment** (what you use during development):

- Run **`langgraph dev`** from the repo root. This starts a LangGraph server on your machine.
- The server loads the graphs defined in **`langgraph.json`** and exposes API endpoints (e.g. run a graph, get state, resume from interrupt).
- **Threads** (checkpoints) are stored **locally** (e.g. in a folder in the project or in a local DB). The **store** (long-term memory) may be a pickled dict or similar on disk.
- **LangGraph Studio** and **Agent Inbox** connect to this local server so you can test the full flow (including interrupts and memory) without deploying to the cloud.

**Hosted deployment** (for production):

- You connect your **GitHub repo** to **LangSmith** (or LangGraph Platform) and create a **deployment**. The platform builds and runs your graph in the cloud.
- **Threads** and **store** are stored in **Postgres** (or another managed DB), so they persist and can be shared across restarts and users.
- You get a **deployment URL** (e.g. `https://your-app.xxx.us.langgraph.app`). You use this URL to run the graph (e.g. from scripts, cron jobs, or Agent Inbox).
- Same code and same `langgraph.json`; only the **environment** (and where checkpointer/store live) changes.

So: **local** = `langgraph dev` + local files; **hosted** = GitHub + Platform + Postgres.

---

## Repo Structure for Deployment

To deploy with LangGraph Platform, your repo typically has:

- **`langgraph.json`** – Config in the repo root: lists **graphs** (name → path to Python module and graph object), **dependencies** (e.g. `.` for the current package), **Python version**, **env** (e.g. `.env`), and optionally dockerfile lines.
- **Graph code** – Python files under e.g. `src/email_assistant/` that define the graph (and optionally the checkpointer/store for local testing). For **hosted** runs, the platform often **injects** the checkpointer and store, so in the **deployable** script you **don’t** pass them when compiling—the platform adds them automatically.
- **Dependencies** – Installed from the repo (e.g. `uv sync` or `pip install -e .`) as specified in `langgraph.json`.

So the same repo works for both `langgraph dev` (local) and “deploy from GitHub” (hosted); the platform reads `langgraph.json` and runs your graphs.

---

## Checkpointer and Store: When to Pass Them

- **In a notebook or local script** – You **do** pass a checkpointer (and optionally a store) when compiling the graph, e.g. for testing interrupts and memory. You control exactly which implementation (e.g. in-memory) is used.
- **In the deployable graph script** (e.g. `email_assistant_hitl_memory_gmail.py`) – You **don’t** pass a checkpointer or store when compiling. LangGraph Platform **supplies** them when it runs your graph: locally they may be file-based/pickled, and in hosted they are Postgres-backed. So the script stays “platform-agnostic”; the environment provides persistence.

This is a small but important point: the deployable code compiles the graph **without** checkpointer/store; the platform attaches them at runtime.

---

## Connecting to Gmail (Overview)

The repo has **mock** tools (default) and **Gmail** tools. To use real Gmail (and Calendar):

1. **Enable APIs** – In Google Cloud Console, enable **Gmail API** and **Google Calendar API** for your project.
2. **OAuth** – Create OAuth credentials (e.g. “Desktop app”), add yourself as a test user if needed, and go through the consent flow. Save the **client secret** JSON to e.g. `tools/gmail/.secrets/`.
3. **Token** – Run the repo’s token-creation script (e.g. once per machine) to get a refresh token. Store it in the same `.secrets` folder or as an env var. The graph (or ingest script) will use this to call Gmail and Calendar.
4. **Graph** – Use the graph that’s wired to **Gmail tools** (e.g. `email_assistant_hitl_memory_gmail`). Same triage + agent + HITL + memory logic; only the tool implementations (send email, check calendar, schedule meeting) switch from mocks to real API calls.

The **Gmail tools** module (e.g. `gmail_tools.py`) defines tools that wrap the Gmail and Calendar APIs; the rest of the agent code stays the same.

---

## Ingesting Emails (Local or Hosted)

To feed **real emails** into your graph:

- **`run_ingest.py`** (or similar) – A script that uses the Gmail API to **fetch** emails (e.g. by address and time range, like “last N minutes”). For each email (or a filtered subset), it calls your graph’s **invoke** (or the deployment API) with that email as input. So “ingest” = fetch from Gmail + run graph once per email.
- **Local** – You run `langgraph dev`, then run the ingest script and point it at `http://127.0.0.1:2024` (or your local server URL). Emails are processed and threads show up in Studio and Agent Inbox.
- **Hosted** – You run the same ingest script but pass the **deployment URL** instead. The script sends requests to the hosted API; threads and store live in Postgres. You can run ingest from your laptop, a server, or a cron job.

So ingest is “pull emails from Gmail → for each, trigger one graph run (local or hosted).”

---

## Cron (Scheduled Ingest)

For a **production** setup you often don’t want to run ingest by hand. The repo includes a **cron** setup (e.g. `setup_cron.py`) that uses the **LangGraph SDK** (or platform API) to register a **scheduled job** for your deployed graph. That job runs on a schedule (e.g. every 10 minutes), and its logic can “fetch recent emails and run the graph for each.” So the platform runs the job; the job uses the Gmail API and your deployment URL. Details (schedule, email filters) are in the script and docs.

---

## Agent Inbox and Hosted Deployment

- **Local** – Agent Inbox connects to your local server URL. It lists interrupted threads that are stored locally and lets you respond (accept, edit, ignore, feedback). When you respond, it sends a **resume** request to the local server with your response payload.
- **Hosted** – You “Add inbox” in the Agent Inbox UI and enter your **deployment URL** (and any API key if required). Inbox then lists interrupted threads from that deployment’s Postgres and sends resume requests to the hosted API. So the same UI works for local and hosted; only the base URL and persistence backend change.

---

## Secrets for Hosted Deployment

When you create a **hosted** deployment in LangSmith/LangGraph Platform, you configure:

- **API keys** – e.g. OpenAI (or other LLM provider) so the graph can call the model.
- **Gmail (and similar)** – You upload or reference **Gmail secret** (client secret JSON) and **Gmail token** (refresh token) as **secrets** in the deployment. The platform injects them as env vars (e.g. `GMAIL_SECRET`, `GMAIL_TOKEN`) so the graph and ingest/cron can authenticate with Gmail and Calendar without you putting secrets in the repo.

So: **local** = `.env` and local files; **hosted** = deployment secrets (and Postgres for state and store).

---

## End-to-End Flow (Hosted + Gmail)

1. **Deploy** – Connect repo to LangGraph Platform, add secrets (OpenAI, Gmail secret, Gmail token), create deployment. You get a URL.
2. **Cron** – Run `setup_cron.py` (or equivalent) with that URL so the platform runs your ingest job on a schedule. The job fetches new emails from Gmail and invokes the graph for each.
3. **Agent** – For each email, the graph runs: triage → maybe response agent → maybe interrupts (e.g. write_email, schedule_meeting). Threads and memory live in Postgres.
4. **You** – Open Agent Inbox, connect it to the deployment URL. You see interrupted threads (e.g. “approve this email”), respond (accept/edit/ignore/feedback). The graph resumes and continues; sent emails go out via Gmail.

So the **same** architecture (triage + agent + HITL + memory) runs locally for development and in the cloud with real Gmail for production; only configuration and persistence backend change.

---

## Summary

- **Local** = `langgraph dev`, local threads/store, Studio and Inbox pointed at local URL.
- **Hosted** = deploy from GitHub, Postgres for threads and store, deployment URL for API and Inbox.
- **Graph script** for deployment usually does **not** pass checkpointer/store; the platform provides them.
- **Gmail** = enable APIs, OAuth, store secret + token, use the Gmail-backed graph and tools; ingest script fetches emails and invokes the graph (local or hosted).
- **Cron** = scheduled job that runs ingest (or similar) against your deployment URL so emails are processed automatically.
- **Agent Inbox** works with both local and hosted; you just point it at the right base URL.

These guides together cover: [basics](01-ambient-agents-and-langgraph-basics.md) → [workflows vs agents](02-workflows-vs-agents.md) → [building the assistant](03-building-the-email-assistant.md) → [evaluation](04-evaluation-and-testing.md) → [HITL](05-human-in-the-loop.md) → [memory](06-memory.md) → [deployment and Gmail](07-deployment-and-gmail.md).
