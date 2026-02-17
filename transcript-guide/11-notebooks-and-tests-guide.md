# Notebooks and Tests in This Repo

This guide explains **what each notebook does**, **how to run the test suite**, and **where test data and evaluators live**. Use it when you want to run, debug, or extend tests and notebooks.

---

## 1. Notebooks: What Each One Covers

The `notebooks/` folder contains Jupyter notebooks that walk through the same concepts as the transcript guides, with runnable code and outputs.

| Notebook | Purpose | Corresponding guide / script |
|----------|---------|-----------------------------|
| **langgraph_101.ipynb** | LangGraph basics: state, nodes, edges, one LLM + one tool, conditional edges. Minimal example. | [01-ambient-agents-and-langgraph-basics.md](01-ambient-agents-and-langgraph-basics.md), `langgraph_101.py` |
| **agent.ipynb** | Building the email assistant: triage router + response agent, state, tools, composing workflow + agent. | [03-building-the-email-assistant.md](03-building-the-email-assistant.md), `email_assistant.py` |
| **evaluation.ipynb** | Evaluation with LangSmith: datasets, Evaluate API, triage and tool-call evals, LLM-as-judge for responses. | [04-evaluation-and-testing.md](04-evaluation-and-testing.md), `eval/` |
| **hitl.ipynb** | Human-in-the-loop: interrupts, Agent Inbox payload, accept/edit/ignore/feedback, message history consistency. | [05-human-in-the-loop.md](05-human-in-the-loop.md), `email_assistant_hitl.py` |
| **memory.ipynb** | Memory: store, namespaces, updating preferences from feedback, reading preferences into the prompt. | [06-memory.md](06-memory.md), `email_assistant_hitl_memory.py` |

**How to use them:** Open in Jupyter or VS Code, run cells in order. Ensure the package is installed (`uv sync --extra dev`) and `.env` is set so API calls work. The notebooks assume you're in the repo root or that the project root is on `sys.path`.

---

## 2. Running the Test Suite

### 2.1 Script tests (email assistant behavior)

Tests that invoke the email assistant and check triage, tool calls, or response quality:

```bash
# From repo root, with .venv active or uv run
python tests/run_all_tests.py --all
```

- **What it does:** Runs pytest for the implementations listed in `run_all_tests.py` (e.g. `email_assistant`). For each implementation it runs `test_response.py` with `--agent-module=<implementation>`. Results are logged to LangSmith (project and experiment name are set in the script).
- **Note:** Only some implementations are included (e.g. `email_assistant`). HITL/memory versions are excluded in the script because the evaluation dataset and test logic don’t yet handle the Question tool and interrupt flow; see comments in `run_all_tests.py`.

To run tests for a **single** implementation:

```bash
python tests/run_all_tests.py --implementation email_assistant
```

### 2.2 Running pytest directly

If you want to run a specific test file or pass extra options:

```bash
# From repo root
python -m pytest tests/test_response.py -v --agent-module=email_assistant

# With LangSmith logging (optional; run_all_tests sets this)
set LANGSMITH_PROJECT=my-project
set LANGSMITH_TEST_SUITE=my-suite
python -m pytest tests/test_response.py -v --langsmith-output --agent-module=email_assistant
```

- **`--agent-module`** is defined in `tests/conftest.py` and is used by `test_response.py` to decide which graph to import and run (e.g. `email_assistant.email_assistant`).

### 2.3 Notebook tests (execute all notebooks)

Check that every notebook runs without errors (no assertions on outputs, just “no exception”):

```bash
python tests/test_notebooks.py
# or
python -m pytest tests/test_notebooks.py -v
```

- **What it does:** Discovers `*.ipynb` under `notebooks/`, runs each with `nbconvert`’s `ExecutePreprocessor` (timeout 600s). If any cell raises an exception, the test fails. Some notebooks can be skipped via `SKIP_NOTEBOOKS` in `test_notebooks.py`.

---

## 3. How the Script Tests Are Wired

### 3.1 conftest.py

- **Purpose:** Shared pytest configuration and fixtures.
- **`pytest_addoption`:** Adds `--agent-module` (default e.g. `email_assistant_hitl_memory`) so you can choose which assistant module to test.
- **Fixture `agent_module_name`:** Reads that option and exposes it to tests. `test_response.py` uses it to import the right graph and build the assistant (with checkpointer/store when needed).

### 3.2 test_response.py

- **Purpose:** End-to-end style tests: run the assistant on example emails and evaluate the response (tool calls and/or LLM-as-judge criteria).
- **Data source:** Imports from `email_assistant.eval.email_dataset`: `email_inputs`, `response_criteria_list`, `expected_tool_calls`, etc. So test cases and expected outputs live in the **eval** package.
- **Flow:** For each test case it (1) builds the assistant (with MemorySaver/InMemoryStore if the module needs them), (2) invokes with the email input, (3) for HITL modules resumes with a fixed “accept” command until done, (4) extracts tool calls and/or final messages and runs evaluators (e.g. criteria grade via an LLM with `CriteriaGrade` schema).
- **LangSmith:** Uses `langsmith.testing` and env vars so runs show up in LangSmith when `--langsmith-output` (or equivalent) is set.

### 3.3 run_all_tests.py

- **Purpose:** Script that sets `LANGSMITH_*` and `LANGCHAIN_TRACING_V2`, then runs pytest once per implementation with the right `--agent-module`. So “run all script tests” = run this file; “run one test file for one agent” = run pytest yourself with `--agent-module=...`.

---

## 4. Where Test and Eval Data Live

| What | Where |
|------|--------|
| **Email examples and expected outputs** | `src/email_assistant/eval/email_dataset.py` (e.g. `email_inputs`, `response_criteria_list`, `expected_tool_calls`, triage examples). |
| **Triage evaluation (dataset + target + evaluator)** | `src/email_assistant/eval/evaluate_triage.py` (creates LangSmith dataset, runs Evaluate API). |
| **Evaluation prompts (e.g. LLM-as-judge)** | `src/email_assistant/eval/prompts.py` (e.g. `RESPONSE_CRITERIA_SYSTEM_PROMPT`). |
| **Test that uses this data** | `tests/test_response.py` imports from `email_assistant.eval.email_dataset` and `email_assistant.eval.prompts`. |

So: to **add a new test case**, add an entry to the structures in `email_dataset.py` (and optionally extend the parametrization in `test_response.py`). To **change how responses are graded**, edit the prompts in `eval/prompts.py` or the schema (e.g. `CriteriaGrade`) in `test_response.py`.

---

## 5. Quick reference

| Goal | Command or location |
|------|----------------------|
| Run all script tests for configured implementations | `python tests/run_all_tests.py --all` |
| Run tests for one implementation | `python tests/run_all_tests.py --implementation email_assistant` |
| Run response tests with a specific module | `python -m pytest tests/test_response.py -v --agent-module=email_assistant` |
| Run notebook tests | `python tests/test_notebooks.py` or `pytest tests/test_notebooks.py -v` |
| Change which agent is tested by default | `tests/conftest.py` → default for `--agent-module` |
| Add or edit test email examples / criteria | `src/email_assistant/eval/email_dataset.py` |
| Edit evaluation prompts (e.g. LLM-as-judge) | `src/email_assistant/eval/prompts.py` |

This should be enough to run the tests, understand how they’re wired, and add or change test data and evaluators in the future.
