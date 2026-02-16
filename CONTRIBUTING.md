# Contributing

## Dependency workflow (uv)

To avoid lock/sync errors when working with this repo:

- **After `git pull`:** run `uv sync --extra dev` so your `.venv` matches the updated `uv.lock`.
- **When you change dependencies:** run `uv lock`, then commit both `pyproject.toml` and `uv.lock`.
- **If you get merge conflicts in `uv.lock`:** resolve `pyproject.toml` first, then run `uv lock` and use the new `uv.lock`.

Full details: [AGENTS.md — Dependency workflow (uv)](AGENTS.md#dependency-workflow-uv--avoid-locksync-errors).
