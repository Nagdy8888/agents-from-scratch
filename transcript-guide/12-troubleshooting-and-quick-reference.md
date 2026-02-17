# Troubleshooting and Quick Reference

This guide covers **common problems** you might hit in this repo and a **one-page quick reference** (commands, paths, env) for future use.

---

## 1. Common issues and fixes

### 1.1 "ModuleNotFoundError: No module named 'email_assistant'" or import errors

**Cause:** The package isn’t installed or the wrong Python/venv is used.

**Fix:**

- From repo root, install the project (and deps) with uv:
  ```bash
  uv sync --extra dev
  ```
- Run your script or tests with the project’s Python:
  ```bash
  uv run python path/to/script.py
  # or
  uv run pytest tests/test_response.py -v
  ```
- If you use an IDE, set the interpreter to the repo’s `.venv` (e.g. `my-agent-project/.venv/Scripts/python.exe` on Windows).

---

### 1.2 "OpenAI API key not found" / "Authentication error"

**Cause:** `.env` is missing or not loaded, or `OPENAI_API_KEY` isn’t set.

**Fix:**

- Copy the template and add your key:
  ```bash
  cp .env.example .env
  # Edit .env and set OPENAI_API_KEY=sk-...
  ```
- Ensure code loads `.env` before any API call (e.g. in the main script or `conftest.py`):
  ```python
  from dotenv import load_dotenv
  load_dotenv(".env")
  ```
- Run from the **project root** so the path `.env` is correct (or use an absolute path to `.env`).

---

### 1.3 LangSmith traces not showing / "LangSmith API key not found"

**Cause:** LangSmith env vars are unset or tracing is off.

**Fix:**

- In `.env` set (optional but useful for tests):
  ```
  LANGSMITH_TRACING=true
  LANGSMITH_API_KEY=your_langsmith_api_key
  LANGSMITH_PROJECT=my-project-name
  ```
- For tests, `run_all_tests.py` sets `LANGSMITH_PROJECT` and `LANGCHAIN_TRACING_V2`; use `--langsmith-output` when running pytest by hand if you want results in LangSmith.
- Create an API key at [smith.langchain.com](https://smith.langchain.com/).

---

### 1.4 "uv.lock" merge conflicts or dependency out of sync

**Cause:** Lockfile or `.venv` doesn’t match `pyproject.toml` (e.g. after a pull or after adding a dependency).

**Fix:**

- **After git pull:** Run `uv sync --extra dev` so `.venv` matches the updated `uv.lock`.
- **After editing pyproject.toml:** Run `uv lock`, then `uv sync --extra dev`, and commit both `pyproject.toml` and `uv.lock`.
- **Merge conflict in uv.lock:** Resolve `pyproject.toml` first, then run `uv lock` and use the new `uv.lock` (don’t hand-edit the lock file).

---

### 1.5 "langgraph dev" fails or "Connection refused" on port 2024

**Cause:** Another process using the port, or the server didn’t start (e.g. missing deps or bad config).

**Fix:**

- Install LangGraph CLI: in `pyproject.toml` add `langgraph-cli[inmem]>=0.4.0`, then `uv lock` and `uv sync --extra dev`.
- Check that `langgraph.json` exists in the repo root and that the graph path (e.g. `./src/email_assistant/agent.py:graph`) points to a real file and variable.
- Try another port: `uv run langgraph dev --port 2025`.
- On Windows, if the browser doesn’t open, use the Studio URL printed in the terminal (e.g. `https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:2024`).

---

### 1.6 Gmail / OAuth errors (e.g. "invalid_grant", "redirect_uri_mismatch")

**Cause:** Wrong OAuth client type, missing test user, or wrong paths for secrets/token.

**Fix:**

- Use a **Desktop app** OAuth client (not Web application) and add your account as a test user if it’s a personal Gmail.
- Put the downloaded JSON in `src/email_assistant/tools/gmail/.secrets/secrets.json`.
- Run `python src/email_assistant/tools/gmail/setup_gmail.py` from repo root; complete the browser flow so `token.json` is created in `.secrets/`.
- For deployment, set `GMAIL_SECRET` and `GMAIL_TOKEN` (or the env names your deployment expects) from the same credentials; see `src/email_assistant/tools/gmail/README.md`.

---

### 1.7 Tests hang or loop (e.g. in test_response.py with HITL agent)

**Cause:** The test assumes a certain interrupt flow (e.g. only “accept”); if the agent uses the Question tool or a different interrupt, the test’s resume logic may not match and it can loop.

**Fix:**

- Use an implementation that matches the test dataset: `run_all_tests.py` only runs implementations that don’t rely on Question-tool handling (e.g. `email_assistant`).
- For HITL/memory agents, the test would need to handle multiple interrupt types (e.g. Question vs write_email) and possibly different resume payloads; see comments in `run_all_tests.py` and logic in `test_response.py`.

---

## 2. Quick reference

### 2.1 Commands (from repo root)

| Task | Command |
|------|--------|
| Install deps and package | `uv sync --extra dev` |
| Regenerate lockfile | `uv lock` |
| Run script tests (all configured implementations) | `python tests/run_all_tests.py --all` |
| Run script tests for one implementation | `python tests/run_all_tests.py --implementation email_assistant` |
| Run notebook tests | `python tests/test_notebooks.py` or `pytest tests/test_notebooks.py -v` |
| Start LangGraph dev server | `uv run langgraph dev` |
| Gmail OAuth setup | `python src/email_assistant/tools/gmail/setup_gmail.py` |
| One-off email ingest (local server) | `python src/email_assistant/tools/gmail/run_ingest.py --email you@example.com --minutes-since 60` |
| Set up cron for ingest | `python src/email_assistant/tools/gmail/setup_cron.py --email you@example.com --url <deployment_url>` |

### 2.2 Important paths

| What | Path |
|------|------|
| Package code | `src/email_assistant/` |
| Graph entrypoints | `src/email_assistant/*.py` (e.g. `email_assistant.py`, `email_assistant_hitl_memory_gmail.py`) |
| Eval data and triage eval | `src/email_assistant/eval/email_dataset.py`, `evaluate_triage.py`, `prompts.py` |
| Default tools | `src/email_assistant/tools/default/` |
| Gmail tools and ingest | `src/email_assistant/tools/gmail/` (README, run_ingest.py, setup_cron.py, setup_gmail.py) |
| Tests | `tests/` (conftest.py, run_all_tests.py, test_response.py, test_notebooks.py) |
| Notebooks | `notebooks/` (langgraph_101, agent, evaluation, hitl, memory) |
| Env template | `.env.example` |
| LangGraph config | `langgraph.json` (root) |

### 2.3 Environment variables (see .env.example)

| Variable | Purpose |
|----------|---------|
| `OPENAI_API_KEY` | Required for OpenAI-backed agents. |
| `LANGSMITH_TRACING` | Set to `true` to send traces to LangSmith. |
| `LANGSMITH_API_KEY` | LangSmith API key (for tracing and experiments). |
| `LANGSMITH_PROJECT` | Project name in LangSmith. |

Optional for tests: `LANGSMITH_TEST_SUITE`, `LANGSMITH_EXPERIMENT` (or let `run_all_tests.py` set them).

---

## 3. Where to look next

- **Notebooks and tests in detail:** [11-notebooks-and-tests-guide.md](11-notebooks-and-tests-guide.md)
- **Dependency workflow (uv, lockfile):** [AGENTS.md](../AGENTS.md) or [CONTRIBUTING.md](../CONTRIBUTING.md) (root)
- **Gmail setup and deploy:** [src/email_assistant/tools/gmail/README.md](../src/email_assistant/tools/gmail/README.md)
- **Starting a new agent project:** [10-starting-a-new-ai-agent-project.md](10-starting-a-new-ai-agent-project.md)

Keeping this file and the quick reference section handy should speed up debugging and day-to-day use of the repo.
