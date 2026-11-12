---
name: backend-dev
description: Implement a backend issue in srmarjani/kagent-todo-app (FastAPI, SQLite, pytest) and open a pull request.
---

# Backend work on kagent-todo-app

1. Read the issue and `README.md` (the spec) with your GitHub tools. Read every file you will change.
2. Plan the change in 2–4 lines before writing anything.
3. Create a branch `issue-<n>-<short-slug>` from `master`.
4. Write complete files: FastAPI in `app/main.py`, SQLite with the standard library `sqlite3` in `app/db.py`,
   DB path from env `TODO_DB_PATH` (default `./todo.db`). Pin versions in `requirements.txt`.
5. Every endpoint change gets a test in `tests/test_api.py` (FastAPI `TestClient`, temporary DB per test).
6. You cannot run code. Re-read each file for imports, names and status codes before committing.
7. Commit all files in one `push_files` call, then open a PR: title from the issue, body with `Closes #<n>`,
   files changed, and how to test (`pip install -r requirements.txt && pytest -q`).
