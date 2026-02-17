# Detailed Steps: Starting a New AI Agent Project

This guide walks you through creating a **new project folder** for designing an AI agent: folder structure, requirements, dependencies, `uv` and `uv.lock`, environment setup, optional Docker, and LangGraph configuration. Follow the steps in order.

---

## Table of contents

1. [Prerequisites](#1-prerequisites)
2. [Create the project folder and basic structure](#2-create-the-project-folder-and-basic-structure)
3. [Set up Python and dependency management (uv)](#3-set-up-python-and-dependency-management-uv)
4. [Define dependencies in pyproject.toml](#4-define-dependencies-in-pyprojecttoml)
5. [Generate and use uv.lock](#5-generate-and-use-uvlock)
6. [Environment variables and .env](#6-environment-variables-and-env)
7. [Optional: LangGraph configuration (langgraph.json)](#7-optional-langgraph-configuration-langgraphjson)
8. [Optional: Docker](#8-optional-docker)
9. [.gitignore and what to commit](#9-gitignore-and-what-to-commit)
10. [Quick checklist](#10-quick-checklist)

---

## 1. Prerequisites

Before starting, have the following installed and ready:

| Tool | Purpose | How to get it |
|------|----------|----------------|
| **Python 3.11+** | Runtime for the agent | [python.org](https://www.python.org/downloads/) or your system package manager |
| **uv** | Fast dependency installer and lockfile manager | `pip install uv` or [install guide](https://github.com/astral-sh/uv#installation) |
| **Git** | Version control | [git-scm.com](https://git-scm.com/) |

Optional (for deployment and observability):

- **Docker** — if you plan to run the agent in a container or use LangGraph’s container-based deployment.
- **LangSmith account** — for tracing and evaluating your agent (sign up at [smith.langchain.com](https://smith.langchain.com/)).

---

## 2. Create the project folder and basic structure

Create a dedicated folder for the agent and a minimal layout so code and config stay organized.

### 2.1 Create the root folder

```bash
mkdir my-agent-project
cd my-agent-project
```

Replace `my-agent-project` with your project name (use lowercase and hyphens for consistency).

### 2.2 Recommended folder structure

A typical layout for an AI agent (e.g. LangChain/LangGraph) looks like this:

```
my-agent-project/
├── .env.example          # Template for env vars (committed)
├── .env                  # Your real secrets (not committed)
├── .gitignore
├── README.md
├── pyproject.toml        # Project metadata and dependencies
├── uv.lock               # Locked dependency versions (committed)
├── langgraph.json        # Optional: for LangGraph dev/deploy
├── src/
│   └── my_agent/         # Main package (Python module)
│       ├── __init__.py
│       ├── agent.py      # Or: graph definition, nodes, etc.
│       ├── prompts.py
│       ├── schemas.py
│       └── utils.py
├── tests/
│   ├── __init__.py
│   └── test_agent.py
└── notebooks/            # Optional: for experiments
    └── .gitkeep
```

**Why `src/`?**  
Code lives under `src/my_agent/` so the installed package is the same as what you run in production (you install the package in “editable” mode and import `my_agent`). This avoids accidentally importing from the repo root.

**Next steps in this guide:** we will create `pyproject.toml`, then `src/my_agent/`, then `.env.example` and `.gitignore`, and optionally `langgraph.json` and Docker.

---

## 3. Set up Python and dependency management (uv)

Using **uv** gives you fast installs and a single lockfile (`uv.lock`) so everyone gets the same dependency versions.

### 3.1 Install uv (if not already)

```bash
pip install uv
```

### 3.2 Ensure the correct Python version

Your project will pin a version in `pyproject.toml` (e.g. `>=3.11,<3.14`). Make sure that Python is available:

```bash
python --version   # or python3
```

If you need multiple Python versions, use `pyenv`, `uv python install 3.11`, or your system’s tooling.

### 3.3 Initialize the project with uv (creates pyproject.toml)

From the **project root** (`my-agent-project/`):

```bash
uv init
```

This creates a minimal `pyproject.toml`. You can then edit it (next section) to add your project name, description, and dependencies. Alternatively, you can skip `uv init` and create `pyproject.toml` by hand.

---

## 4. Define dependencies in pyproject.toml

`pyproject.toml` is the single source of truth for project metadata and dependencies. Do **not** use `requirements.txt` when using uv; uv uses `pyproject.toml` and generates `uv.lock`.

### 4.1 Minimal pyproject.toml for an AI agent

Create or replace `pyproject.toml` in the project root with something like the following. Adjust the project name and package name to match your folder (e.g. `my_agent` under `src/my_agent/`).

```toml
[project]
name = "my-agent-project"
version = "0.1.0"
description = "Short description of your AI agent"
requires-python = ">=3.11,<3.14"
dependencies = [
    "langchain>=1.0.0",
    "langchain-core>=1.0.0",
    "langchain-openai>=1.0.0",
    "langgraph>=1.0.0",
    "langsmith>=0.4.37",
    "python-dotenv>=1.0.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0.0",
    "mypy>=1.11.0",
    "ruff>=0.6.0",
]

[build-system]
requires = ["setuptools>=73.0.0", "wheel"]
build-backend = "setuptools.build_meta"

[tool.setuptools.packages.find]
where = ["src"]

[tool.setuptools.package-dir]
"" = "src"
```

**Explanation of key fields:**

| Section | Purpose |
|--------|---------|
| `[project]` | Name, version, description, **Python version range**, and **list of dependencies**. |
| `requires-python` | Pins the Python version (e.g. 3.11+ and below 3.14). uv and pip will enforce this. |
| `dependencies` | Runtime dependencies. Add others as needed (e.g. `langgraph-cli[inmem]` for `langgraph dev`). |
| `[project.optional-dependencies]` | Optional groups; e.g. `dev` for testing and linting. Install with `uv sync --extra dev`. |
| `[build-system]` | Tells pip/uv how to build the package (setuptools). |
| `[tool.setuptools.*]` | Tells setuptools that packages live under `src/` so `my_agent` is found as a top-level package. |

### 4.2 If your package name differs from the folder name

If your code lives in `src/email_assistant/` but you want the installable package to be named `my_agent`, you can do:

```toml
[tool.setuptools]
packages = ["my_agent"]

[tool.setuptools.package-dir]
"my_agent" = "src/email_assistant"
```

Then you would import with `from my_agent import ...`. For simplicity, we assume the folder under `src/` is the package name (e.g. `src/my_agent/` → `import my_agent`).

### 4.3 Adding more dependencies later

- **Add a new dependency:** put it in `pyproject.toml` under `dependencies` (or under `[project.optional-dependencies].dev`), then run `uv lock` (see next section).
- **Do not** edit `uv.lock` by hand; always regenerate it with `uv lock`.

---

## 5. Generate and use uv.lock

The lockfile fixes **exact versions** of every dependency (and their sub-dependencies) so installs are reproducible.

### 5.1 Generate the lockfile

From the project root:

```bash
uv lock
```

This creates or updates `uv.lock`. Commit `uv.lock` to Git so that everyone (and CI) uses the same versions.

### 5.2 Install dependencies into a virtual environment

Create a venv and install the locked dependencies:

```bash
uv sync
```

To include optional “dev” dependencies (e.g. pytest, ruff):

```bash
uv sync --extra dev
```

`uv sync` will create a `.venv` in the project root if it doesn’t exist and install from `uv.lock`. After this, use this project’s Python and scripts via:

- `uv run python ...` or  
- Activate `.venv` and run `python` / `pytest` as usual.

### 5.3 Daily workflow

| Situation | Command |
|----------|--------|
| You added or changed a dependency in `pyproject.toml` | `uv lock` then `uv sync` (or `uv sync --extra dev`). Commit `pyproject.toml` and `uv.lock`. |
| You pulled changes that touch `pyproject.toml` or `uv.lock` | `uv sync` (or `uv sync --extra dev`) to refresh `.venv`. |
| You want to bump or add a dependency | Edit `pyproject.toml`, run `uv lock`, then `uv sync`. |

Never commit `.venv`; always commit `uv.lock`.

---

## 6. Environment variables and .env

Agents usually need API keys and optional settings. Keep these out of the code and out of Git by using a `.env` file and loading it with `python-dotenv`.

### 6.1 Create .env.example (committed)

Create a **template** that lists required and optional variables **without** real secrets:

```bash
# .env.example

# Required for OpenAI-backed agents
OPENAI_API_KEY=your_openai_api_key_here

# Optional: LangSmith tracing and evaluation
LANGSMITH_TRACING=false
LANGSMITH_API_KEY=your_langsmith_api_key_optional
LANGSMITH_PROJECT=my-agent-project
```

Commit `.env.example` so others know what to set.

### 6.2 Create .env (not committed)

Copy the template and fill in real values **only on your machine**:

```bash
cp .env.example .env
# Then edit .env and add your real API keys.
```

Add `.env` to `.gitignore` (see below) so it is never committed.

### 6.3 Load .env in code

In your application or test setup, load the env file as early as possible (e.g. in the main entrypoint or in `conftest.py`):

```python
from dotenv import load_dotenv
load_dotenv(".env")
```

Then use `os.environ["OPENAI_API_KEY"]` or a small config module that reads from `os.environ`. Do not put default API keys in code.

---

## 7. Optional: LangGraph configuration (langgraph.json)

If your agent is built with **LangGraph** and you want to run it with `langgraph dev` or deploy it with LangGraph Platform, add a `langgraph.json` in the project root.

### 7.1 Minimal langgraph.json

```json
{
  "graphs": {
    "my_agent": "./src/my_agent/agent.py:graph"
  },
  "python_version": "3.11",
  "env": ".env",
  "dependencies": ["."],
  "dockerfile_lines": []
}
```

**Meaning of fields:**

| Field | Purpose |
|-------|--------|
| `graphs` | Map of graph **name** → **path to module and variable**. The path is `./path/to/file.py:variable_name` where `variable_name` is the compiled LangGraph in that file (e.g. `graph` or `app`). |
| `python_version` | Python version the platform uses to run the graph. |
| `env` | Path to the env file (e.g. `.env`) loaded when running the server. |
| `dependencies` | How to install the project; `["."]` means “install the current directory as a package” (uses `pyproject.toml`). |
| `dockerfile_lines` | Optional list of Dockerfile lines if you customize the image; can be empty. |

### 7.2 Making sure the graph is exposed

In `src/my_agent/agent.py` you must have a compiled graph assigned to the variable you referenced in `langgraph.json` (e.g. `graph`):

```python
from langgraph.graph import StateGraph, START, END

# ... define state, nodes, edges ...

graph = workflow.compile()
```

Then `langgraph dev` will discover and serve it under the name `my_agent`.

### 7.3 Dependencies for local LangGraph dev

To run `langgraph dev`, you need the LangGraph CLI. Add to `pyproject.toml`:

```toml
dependencies = [
    # ... your existing deps ...
    "langgraph-cli[inmem]>=0.4.0",
]
```

Then `uv lock` and `uv sync --extra dev`. Run from the project root:

```bash
uv run langgraph dev
```

---

## 8. Optional: Docker

You can add Docker later for running the agent in a container or for LangGraph’s container-based deployment. The following is a minimal pattern.

### 8.1 When to add Docker

- You want to run the agent in a container locally or in the cloud.
- Your deployment platform expects a Dockerfile (e.g. some LangGraph deployment options).

If you only use `langgraph dev` and a hosted LangGraph Platform that builds the image for you, you may not need your own Dockerfile.

### 8.2 Minimal Dockerfile pattern

Create a `Dockerfile` in the project root:

```dockerfile
# Use the same Python version as in pyproject.toml
FROM python:3.11-slim

WORKDIR /app

# Install uv (or use a base image that has it)
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv

# Copy dependency manifests first (better layer caching)
COPY pyproject.toml uv.lock ./

# Install dependencies (no dev extras)
RUN uv sync --no-dev --no-install-project

# Copy the rest of the app
COPY . .

# Install the project itself
RUN uv sync --no-dev --no-install-project --package .

# Run the app (adjust to your entrypoint)
# CMD ["uv", "run", "python", "-m", "my_agent.main"]
CMD ["uv", "run", "langgraph", "dev"]
```

Adjust the `CMD` to whatever starts your agent (e.g. a FastAPI app or `langgraph dev`). If you use `langgraph.json`, ensure the Docker build copies it and that `dependencies` and `env` are correct for the container.

### 8.3 .dockerignore

Create a `.dockerignore` so the build doesn’t send secrets or unnecessary files into the image:

```
.venv
.env
.git
__pycache__
*.pyc
.pytest_cache
.coverage
*.egg-info
```

---

## 9. .gitignore and what to commit

### 9.1 Minimal .gitignore

Create or update `.gitignore` in the project root:

```gitignore
# Environment and secrets
.env
.env.local
.secrets/

# Python
__pycache__/
*.py[cod]
*.so
.Python
.venv
venv/
env/
*.egg-info/
dist/
build/

# IDE and OS
.idea/
.vscode/
.DS_Store

# Testing and tools
.pytest_cache/
.coverage
htmlcov/
.mypy_cache/
.ruff_cache/
```

Add any other paths you want to keep local (e.g. `notebooks/*.ipynb_checkpoints`, `*.log`).

### 9.2 What to commit

| Commit | Do not commit |
|--------|----------------|
| `pyproject.toml` | `.env` (secrets) |
| `uv.lock` | `.venv/` (recreate with `uv sync`) |
| `.env.example` | `*.egg-info/`, `__pycache__/` |
| `langgraph.json` (if used) | Local IDE/OS files if you prefer |
| All source under `src/` and `tests/` | |
| `README.md`, `.gitignore`, optional `Dockerfile` | |

---

## 10. Quick checklist

Use this to verify you didn’t skip a step.

- [ ] Project root folder created (e.g. `my-agent-project/`).
- [ ] `src/<package_name>/` created with at least `__init__.py` and one module (e.g. `agent.py`).
- [ ] `pyproject.toml` created with correct `name`, `requires-python`, and `dependencies`; setuptools configured to find packages under `src/`.
- [ ] `uv lock` run; `uv.lock` present and committed.
- [ ] `uv sync` (or `uv sync --extra dev`) run; `.venv` exists and is in `.gitignore`.
- [ ] `.env.example` created and committed; `.env` created locally and filled with real values; `.env` in `.gitignore`.
- [ ] In code, `load_dotenv(".env")` is called before using any env-based config.
- [ ] If using LangGraph: `langgraph.json` in root, graph variable exposed in the referenced file; `langgraph-cli[inmem]` in dependencies; `uv run langgraph dev` works.
- [ ] `.gitignore` includes `.env`, `.venv`, `__pycache__`, and other local artifacts.
- [ ] Optional: `Dockerfile` and `.dockerignore` added if you need containerized runs.

After this, you can focus on implementing the agent logic (state, nodes, tools, prompts) inside `src/my_agent/` and run or deploy it using `uv run`, `langgraph dev`, or Docker as needed.
