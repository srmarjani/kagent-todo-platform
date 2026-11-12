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


## 2026-11-10 — Phase 1, step 8 (Session + invoke)

- No Anthropic credits; SRE gave an OpenRouter token. Key in `.env` as `OPENROUTER_API_KEY`,
  Secret `kagent-openrouter` created from stdin (`printf '%s' ... | --from-file=KEY=/dev/stdin`), never in a flag or YAML.
- New ModelConfig `openrouter-haiku-model-config`: provider `OpenAI`, `openAI.baseUrl: https://openrouter.ai/api/v1`,
  model `anthropic/claude-haiku-5.5` (dot, not dash). `my-first-template` now points at it.
- Mistake: `baseURL` instead of `baseUrl`. `kubectl apply --dry-run=server` caught it (strict decoding).
  Lesson: server dry-run before apply.
- Mistake: tried `session list --agent`, flag does not exist. Lesson: `--help` before guessing flags.
- Old Session 01a1c23c… still called api.anthropic.com (credit error). New Session 01a1c46e… (revision b2ed2958…)
  answered through OpenRouter and remembered its first message. Check passed.

### Concepts
- Revision: the compiled agent (template + model + harness). A Session is pinned to the revision it was created on
  (`preparedRevision`). Template changes only reach new Sessions.
- Actor: not a Kubernetes object. One gVisor sandbox per Session (`session-<id>`) inside the WorkerPool pod `kagent-default`.
- After each task the Actor is checkpointed and stopped. The conversation lives on its disk (`/data`), not in memory,
  and is restored on the next invoke.
- Each invoke is one Task inside the Session.
- Session state `READY` even while the Actor is stopped: READY means "can take work", not "running".

### Incident: cluster down
- Old clusters were running again; `todo-agents` node exited 137 and was left with no Docker network
  (kind: "old IPv4 but no current IPv4", exit 1 on start).
- Fix: stop old clusters, `docker network connect kind todo-agents-control-plane`, `docker start`.
  All pods and the agent came back without re-applying anything. Who detached it: unknown.

### Decisions
- kagent v1.0.0 is out (same commit as rc4, small diff from rc3). Upgrade with Helm now that step 8 works.
- Concepts first; details go to "Deep dives for later" in LEARNING_PLAN.md.

- What surprised me: ...
- What I still do not understand: ...

### Open
- Upgrade to 1.0.0.
- UI :8082, A2A card, Claude Code via /mcp, prompt change.

## 2026-11-10 / 2026-11-12 — Phase 2: agentgateway in front of the LLM

- Plan switched to agentgateway v1.7.0 (still the newest release) with the model-centric `AgentgatewayModel`
  and its built-in `Openrouter` provider, instead of `AgentgatewayBackend` + `HTTPRoute`.
- Installed Gateway API CRDs v1.6.3 (already present, probably from the Substrate install) and agentgateway
  v1.7.0 with Helm. Helm's flag is `--kube-context`, not `--context`.
- Gateway `ai-gateway`: the listener must list `AgentgatewayModel` in `allowedRoutes.kinds`, otherwise an HTTP
  listener only accepts `HTTPRoute`. The controller starts a proxy Deployment and Service with the Gateway's name.
- Secret `openrouter-secret` (key `Authorization`, raw key) in `agentgateway-system`, from `.env` via stdin.
  v1.7.0 adds `Bearer ` itself and strips one if present (`kubectl explain agentgatewaymodel.spec.policies.auth`).
- `AgentgatewayModel` `openrouter-haiku`: `match.model: anthropic/claude-haiku-5.5`. Test via port-forward:
  `/v1/models` lists only Haiku; a chat request with no key in it worked; Sonnet got 404 `model_not_found`.
- New ModelConfig `openrouter-via-gateway` (provider OpenAI, `baseUrl` = gateway `/v1`, dummy key Secret).
  Template switch made revision 3ba40435…; new Session 01a1ceb2… answered like a pirate.
- Deleted the Phase 1 direct ModelConfig + `kagent-openrouter`, and the Helm-owned `default-model-config` +
  `kagent-anthropic` (Helm may recreate those on upgrade, noted in Phase 8). Check passed: `kagent` has only
  the dummy key.

### Concepts
- Two gateways: `atenet-egress` decides where a sandbox may connect; `ai-gateway` decides which model, with
  which key, and logs tokens per request. Path: Actor → atenet-egress → ai-gateway → openrouter.ai.
- No EgressPolicy needed: kagent inferred the in-cluster gateway from the ModelConfig (find-out answered).
- In Phase 1 the sandbox never held the key either: the egress gateway injected it from the Secret. An old
  Session (revision a5605edb…) now fails with 403 "atespace kagent is not permitted to resolve secret".
- What Phase 2 adds: one owner for LLM keys, a model allow-list, LLM-aware logs, one place for budgets later.
- `ai-gateway` sees the egress pod as the client, not the agent: per-agent limits need the identity carried
  along (ModelConfig `actorJwt`? Phase 6).
- A "hi" costs ~2,200 input tokens: system prompt, tools and framing go with every call.

### Decisions
- From 2026-11-12 Claude writes files and runs commands; I read and approve each step (CLAUDE.md).

### Open
- Teach-back for Phase 2.
- Which kagent Helm values stop the chart from recreating the Anthropic key (Phase 8).
