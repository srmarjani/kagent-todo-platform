---
name: devops
description: Build, ship and deploy srmarjani/kagent-todo-app (Dockerfile, GitHub Actions image to GHCR, Kubernetes manifests in namespace todo).
---

# DevOps work on kagent-todo-app

1. Read the issue and `README.md` (the spec) with your GitHub tools. Read every file you will change.
2. Plan the change in 2–4 lines before writing anything.
3. Create a branch `issue-<n>-<short-slug>` from `master`.
4. `Dockerfile` and `.github/workflows/build.yml` exactly as the README describes: run tests, then push
   `ghcr.io/srmarjani/kagent-todo-app:<sha>` and `:latest` with `GITHUB_TOKEN` (`permissions: packages: write`).
5. `k8s/` manifests: namespace `todo`, readiness probe on `/healthz`, Service NodePort `30080` to port 8000.
   Deployments use an exact `:<sha>` image tag, never `:latest`.
6. Commit all files in one `push_files` call, then open a PR: title from the issue, body with `Closes #<n>`,
   files changed, and how to test.
7. Apply to the cluster only with `k8s_apply_manifest`, only after the PR is merged, and only in namespace `todo`.
   Afterwards check the rollout and the pod logs and report what you saw.
8. Never touch other namespaces. Never read, list or describe Secrets.
