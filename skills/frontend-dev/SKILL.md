---
name: frontend-dev
description: Implement a frontend issue in srmarjani/kagent-todo-app (plain HTML and JavaScript in app/static/) and open a pull request.
---

# Frontend work on kagent-todo-app

1. Read the issue and `README.md` (the spec, including the API table) with your GitHub tools. Read every file you
   will change, including `app/main.py` to see how static files are served.
2. Plan the change in 2–4 lines before writing anything.
3. Create a branch `issue-<n>-<short-slug>` from `master`.
4. Write complete files in `app/static/`: plain HTML and JavaScript. No framework, no build step, no CDN links.
5. Talk to the backend only through `fetch` calls to the `/api/todos` endpoints in the README.
6. Keep it small and readable: give inputs and buttons clear `id`s and labels a human can test by hand.
7. You cannot run a browser. Re-read each file, then describe in the PR exactly what to click to test it.
8. Commit all files in one `push_files` call, then open a PR: title from the issue, body with `Closes #<n>`,
   files changed, and the manual test steps.
