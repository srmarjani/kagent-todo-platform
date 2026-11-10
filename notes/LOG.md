## 2026-11-09 — Phase 0

- First time with the GitHub CLI (`gh`). Created `kagent-todo-platform` and `kagent-todo-app`.
- Fine-grained PAT scoped to `kagent-todo-app` only (Contents, Issues, PRs). Keys live in `.env` (600, gitignored).
- Pinned versions in `versions.env`: kagent v1.0.0-rc3, Substrate v1.0.0, node image v1.37.0,
  agentgateway v1.7.0, agentregistry v0.4.0.
- Stopped the two old 0.10.4 kind clusters to free memory (`docker stop`, reversible).
- Mistake: cloned the app repo inside the platform folder, moved it to `~/Tutorial/kagent-todo-app`.
- Mistake: saved `versions.env` and `kind-config.yaml` empty; `yq` returned 0/null. Lesson: check the file after writing.

## 2026-11-10 — Phase 1, install

- Cluster `todo-agents` created. `podcertificaterequests` present in `certificates.k8s.io` (prerequisite OK).
- Config mistakes I made: `provider:` instead of `providers:` in the Helm values
  (`kagent config validate` did not catch it, it only validates the CLI config);
  `defaultModelProvider` under `install.kagent` instead of `install`.
- `kagent install` needed three runs:
  1. Substrate Helm wait exceeded. Images from ghcr.io pulled slowly (1–4 min each),
     `atenet-router` ImagePullBackOff from `cr.agentgateway.dev`. Pods healed on their own later.
  2. Substrate OK, kagent release failed after exactly 5 min.
  3. "kagent installed successfully", kagent release revision 2.
- Inventory after install: Helm releases kagent, kagent-crds (ns kagent), substrate, substrate-crds,
  substrate-atelet (ns ate-system), substrate-podcert (ns podcertificate-controller-system).
  Pre-registered RemoteMCPServers: `kagent-api` (controller /mcp) and `kagent-tool-server`.
  ModelConfig `default-model-config`. WorkerPool `kagent-default`.

### Find-out answers

1. CLI config key for the default provider: `install.defaultModelProvider` (PascalCase, e.g. `Anthropic`).
   Helm values use `providers.default` (camelCase, e.g. `anthropic`). Two different files, two different spellings.
2. `kagent install --timeout` does not extend the per-chart Helm waits (~10 min Substrate, ~5 min kagent).
3. The CLI re-installs a failed Helm release on the next run; no `helm uninstall` needed.
4. `cr.agentgateway.dev` was flaky from my network. `ghcr.io/agentgateway/agentgateway:v1.7.0` has the same digest.
   Override for Substrate if it happens again: values key `images.agentgateway`.
5. Actors are not Kubernetes objects. `ate.dev` has only csidriverconfigs, sandboxconfigs, workerpools.
   Sessions/Actors live in Postgres behind `ate-api-server`. Still to check: `kagent sandbox list` or `kubectl-ate`.

## 2026-11-10 — Phase 1, first agent

- Harness `my-first-harness`, AgentTemplate `my-first-template`, Agent `my-first-agent`, all READY True.
- Mistake: Harness was named `harness` but the Agent referenced `my-first-harness`.
  Condition chain showed it: Accepted → EgressPoliciesApplied → ResolvedRefs=False (ReferenceResolutionFailed).
  Lesson: read `.status.conditions` first, the reason names the broken field.
- No pod per agent: `kubectl get pods -A | grep my-first` is empty. Proven.
- Session created: 01a1c23c-c7db-7b80-8435-3389e9fc7d0d.
- Invoke failed with Anthropic 400 "credit balance is too low". Key is identical in .env, Secret and shell,
  so the whole path Session → egress → Anthropic works; only billing is missing.
- What surprised me: ...
- What I still do not understand: ...

### Open

- Add Anthropic credits, retry invoke on the same Session, then a follow-up message.
- Which client shows the live Actor, and what an idle Session looks like.
- UI :8082, A2A agent card path, Claude Code via /mcp, prompt change → old vs new Session.