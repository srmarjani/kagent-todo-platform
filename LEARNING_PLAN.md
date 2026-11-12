# Learning plan: kagent 1.x, agentgateway and agentregistry, by building a todo app with an agent team

Written 2026-11-09 for Razi. Replaces `~/Tutorial/kagent_tutorial/LEARNING_PLAN.md`.

**The idea.** One kind cluster. kagent 1.x runs four agents: `team-lead`, `backend-dev`, `frontend-dev`, `devops`.
They build a small FastAPI + HTML/JS todo app in a GitHub repo through pull requests you review, and deploy it into
the same cluster. agentgateway sits between the agents and the LLM (Anthropic models through OpenRouter).
agentregistry catalogs the tools, skills and agents. Since 2026-11-12 Claude writes the files and runs the commands,
and you read and approve each step first (rules in `CLAUDE.md`).

## How to use this plan

- Work phase by phase. The **Check** and **Teach-back** blocks are the learning; do not skip them.
- Keep `notes/LOG.md`: after each session write what worked, what surprised you, open questions.
  Keep `notes/GLOSSARY.md` in your own words.
- **verified** = checked against the 1.x docs or a release page on 2026-11-09.
  **find out** = the docs did not answer it; you discover it and write the answer in the log.
- kagent `v1.0.0` (GA) was published on 2026-11-09, after Phase 1 had started on `v1.0.0-rc3`. Decision
  (2026-11-10): stay on rc3 through all phases and upgrade in Phase 8. Upgrade earlier only if an rc3 bug blocks
  you. The likely ones: MCP tool discovery when a Session resumes (Phase 3, fixed in 1.0.0 by
  https://github.com/kagent-dev/kagent/pull/3560) and `invoke_agent` (Phase 4.3).
- Each step: Claude explains it and shows the file or command, you approve, Claude runs it and explains the result.
  You can still type any step yourself.

## Effort (part time, hands-on hours)

| Phase | Topic | Hours |
|---|---|---|
| 0 | Prepare machine, repos, secrets | 2–3 |
| 1 | kind + kagent 1.x alone: first agent, sessions, A2A | 4–6 |
| 2 | agentgateway in front of the LLM | 3–4 |
| 3 | Tools: GitHub MCP, approvals, first useful agent | 4–6 |
| 4 | The team builds the todo app | 12–16 |
| 5 | agentregistry as the catalog | 4–6 |
| 6 | Gateway governance + traces (optional) | 4–6 |
| 7 | Claude Code harness comparison (optional) | 3–4 |
| 8 | Retro | 1 |

Core (0–5): about 30–40 h. With optional phases: about 45 h.

## Pinned versions (copy into `versions.env`)

| Component | Version | Notes |
|---|---|---|
| kind / node image | v0.33.0 / `kindest/node:v1.37.0` | already installed / pulled (**verified**) |
| kagent CLI + charts | `v1.0.0-rc3` | `v1.0.0` is out; upgrade in Phase 8 (decision 2026-11-10) |
| Agent Substrate | `v1.0.0` (installed by the kagent CLI) | worker image `ateworker-gvisor:v1.0.0` |
| kagent-tools | 0.3.0 | bundled `kagent-tool-server` |
| Gateway API CRDs | v1.6.3 | standard channel (re-checked 2026-11-10) |
| agentgateway | v1.7.0 (latest, re-checked 2026-11-10) | charts at `oci://cr.agentgateway.dev/charts/`; LLM via `AgentgatewayModel` |
| agentregistry | v0.4.0 (`arctl` same) | 0.3.3 is the only version known to work with kagent **0.x** deployments; see Phase 5 |
| Models (via OpenRouter) | `anthropic/claude-haiku-5.5` for learning, Sonnet 5.5 for coding agents | cost control; check the Sonnet ID on openrouter.ai |

## Rules for the whole project

- Always pass `--context kind-todo-agents` to `kubectl` and `helm`. Your kubeconfig holds company clusters.
- Keys live in `.env` (mode 600, gitignored) and reach the cluster through a pipe into a Secret.
  Never in YAML, logs, chat or `--set` flags.
- Never guess a CRD field: `kubectl explain agenttemplate.spec --api-version=api.kagent.dev/v1alpha3`
  (also `harness`, `agent`, `remotemcpserver`, `egresspolicy`).
- Your own resources are plain YAML in this repo. Helm only for third-party charts.
- After you change an AgentTemplate, **create a new Session**; old Sessions keep the old revision (**verified**).
- Port 8083 (controller gRPC/MCP) has `auth.mode: insecure`. Never expose it outside the cluster (**verified**).
- In zsh, quote words starting with `=`.

## Repo layout (this folder, GitHub repo `kagent-todo-platform`)

```
LEARNING_PLAN.md  CLAUDE.md  versions.env  .env (gitignored)
cluster/    kind-config.yaml
kagent/     kagent-config.yaml, kagent-values.yaml
gateway/    gateway.yaml, openrouter-model.yaml, (later) sonnet model, policies
agents/     00-harness.yaml, 01-hello.yaml, 10-modelconfigs.yaml, 20-mcp-github.yaml,
            21-mcp-kagent.yaml, 30-backend-dev.yaml, 31-frontend-dev.yaml, 32-devops.yaml,
            40-team-lead.yaml, 50-egress.yaml
skills/     backend-dev/SKILL.md + Dockerfile, frontend-dev/, devops/
registry/   ar.dev manifests
notes/      LOG.md, GLOSSARY.md, RETRO.md
```

Second GitHub repo `kagent-todo-app` (public, so skills/images need no pull secrets): the app the agents write.

## What you are learning, in one picture

```
you ──(UI :8082 / CLI / Claude Code MCP)──▶ kagent-controller :8083 ──▶ Session (Actor in gVisor sandbox,
                                                                          on WorkerPool kagent-default)
                                                                            │ default-deny egress
                                                                            ▼
                                                     atenet-egress ──▶ ai-gateway (agentgateway) ──▶ OpenRouter
                                                                   ──▶ GitHub MCP (api.githubcopilot.com)
                                                                   ──▶ kagent-tool-server (k8s_* tools)
                                                                   ──▶ kagent-controller /mcp (invoke_agent)
agentregistry: catalog of the MCP servers, skills (OCI images) and agents above
```

---

## Phase 0: Prepare (2–3 h)

Goal: a clean machine, pinned versions, secrets outside git, two repos you can push to.

Do
- [x] Free memory (14 GB total, both old clusters use ~3 GB). Reversible stop:
      ```bash
      docker stop $(docker ps -q --filter name=kagent-oss-demo) $(docker ps -q --filter name=multi-agent-demo)
      ```
      `docker start <names>` brings them back if you want the 0.10.4 reference later.
- [x] Install the kagent CLI at the chart version (**verified**: the docs require matching versions).
      Read the script first, then:
      ```bash
      curl https://raw.githubusercontent.com/kagent-dev/kagent/refs/heads/main/scripts/get-kagent | bash -s -- --version v1.0.0-rc3
      ```
- [x] Install `gh` (GitHub CLI) and `yq`. Optional but they save time. `gh auth login`.
- [x] GitHub: create `kagent-todo-platform` (this folder) and `kagent-todo-app` (empty, public, with README).
      Create a **fine-grained PAT** scoped to `kagent-todo-app` only: Contents RW, Pull requests RW, Issues RW,
      Metadata R. Put it in `.env` as `GITHUB_PAT`. Put your Anthropic key there as `ANTHROPIC_API_KEY`.
      `chmod 600 .env`.
- [x] `git init` this folder, add `.gitignore` (`.env`, `.tmp/`), `versions.env`, `notes/LOG.md`, push.
- [x] Write `cluster/kind-config.yaml`: one control-plane node, image `kindest/node:v1.37.0`,
      `extraPortMappings` `30080 -> 127.0.0.1:8080` (todo app NodePort, Phase 4) and
      `30121 -> 127.0.0.1:12121` (agentregistry, Phase 5).
      Model it on `~/Tutorial/webinar-demos/02-multi-agent-skills/scripts/01-setup-kind.sh`.

Check
- [x] `kagent version` prints the pinned version. `free -g` shows at least 6 GB available.
      `git remote -v` shows both repos (platform here, app in its own clone).

Find out
- Does Helm 4 matter? The kagent CLI does its own installing; your `helm` (v4) is only used for the
  agentgateway and agentregistry charts. Note any failure.

---

## Phase 1: kind + kagent 1.x alone (4–6 h)

Goal: understand the 1.x resource model by touching every piece once. No gateway yet.

Read (in this order)
- https://kagent.dev/docs/kagent/1.x/about/core-concepts/
- https://kagent.dev/docs/kagent/1.x/reference/release-notes/1.0/ (sections "What changed" and "Resource model")
- https://kagent.dev/docs/kagent/1.x/setup/installation/ (CLI path)
- https://kagent.dev/docs/kagent/1.x/get-started/your-first-agent/

Do
- [x] `kind create cluster --name todo-agents --config cluster/kind-config.yaml`.
- [x] Prerequisite (**verified**): `kubectl --context kind-todo-agents api-resources --api-group=certificates.k8s.io`
      lists `podcertificaterequests`.
- [x] `kagent/kagent-values.yaml`: `providers.default: anthropic`, `providers.anthropic.model: claude-haiku-5-5`,
      `config.server.grpcReflection: true`. `kagent/kagent-config.yaml`: `install.kagent.valuesFiles: [kagent-values.yaml]`.
      Check the schema with `kagent config schema` before guessing keys (**find out**: the exact key for the default
      provider in the CLI config vs. Helm values; the docs show `providers.default` in Helm values and the webinar
      rewrite used `install.defaultModelProvider`).
- [x] Export the key from `.env` without echoing it, then install:
      ```bash
      set -a; source .env; set +a; kagent --config kagent/kagent-config.yaml install --namespace kagent
      ```
      Do **not** pre-create `kagent-anthropic`; the installer creates it (**verified**).
- [x] Inventory what the CLI did and write the list in the log: `helm list -A`, namespaces (`kagent`, `ate-system`),
      `kubectl get crds | grep -E 'kagent|ate'`, `kubectl get workerpool -n kagent`,
      `kubectl get modelconfig,remotemcpserver -n kagent`, pods in `ate-system` (`rustfs-bucket-init` should be `Completed`).
- [x] Apply your first Harness (`spec.kagent: {}`, `substrate.workerPoolRef.name: kagent-default`,
      `snapshotPolicy.location: s3://ate-snapshots/kagent/`), an AgentTemplate `hello`
      (`modelConfig.name: default-model-config`, a one-line system prompt), and an Agent binding them
      (`spec.template.ref.name`, `spec.harnessRef.name`). Watch `kubectl get agent -n kagent -w` until `READY True`
      (first time about 1 min: golden snapshot, **verified**).
- [x] Prove there is no pod per agent: `kubectl get pods -A | grep hello` returns nothing.
- [x] Session + invoke:
      ```bash
      SESSION_ID=$(kagent agent session create --agent hello -o json | jq -r '.session.id')
      kagent agent invoke --session "$SESSION_ID" --task "Say hello in one line"
      ```
      Send a second message; confirm it remembers the first (state survived suspend). Watch `ate-system` resources
      during and after the invoke (`kubectl api-resources | grep -i ate`).
      **Find out**: which object is the running Actor and what it looks like once suspended.
- [x] UI: `kubectl port-forward -n kagent svc/kagent-ui 8082:8080` then http://localhost:8082. Chat with `hello` there.
- [x] A2A card: port-forward `svc/kagent-controller 8083:8083`, then
      `curl -s localhost:8083/agents/kagent/hello/.well-known/agent-card.json | jq .`
      (agents live at `/agents/{namespace}/{name}` per the release notes; the exact card path is a **find out**).
- [x] Talk to your agent from Claude Code: `claude mcp add --transport http kagent http://localhost:8083/mcp --scope project`,
      then ask Claude Code to list agents and invoke `hello` (**verified** tools: `list_agents`, `invoke_agent`).
- [x] Change the system prompt, re-apply, invoke on the **old** session (unchanged) and on a **new** session (changed).
      This is the revision model.

Check
- [x] `kagent agent get hello` shows READY True; two sessions behave differently after the prompt change.

Teach-back (write in `notes/GLOSSARY.md`, two sentences each): Harness, AgentTemplate, Agent, Session, Actor,
WorkerPool, golden snapshot, revision, ModelConfig.

Pitfalls (**verified**)
- `READY False` right after apply is normal for about a minute. Still False: run
  `kubectl get agent hello -n kagent -o jsonpath='{.status.conditions}'` (Accepted, ResolvedRefs, Compatible, Ready).
- The bundled Postgres has no pgvector, so long-term memory features are off. Fine for this plan.
- The bundled RustFS object store is not durable (1 Gi PVC). Fine for a laptop.

---

## Phase 2: agentgateway in front of the LLM (3–4 h)

Goal: the OpenRouter key leaves kagent and lives in the gateway; one place to see, route and later limit LLM traffic.

Updated 2026-11-10 for the latest agentgateway. v1.7.0 is still the newest release, and the "latest" docs are the
1.7.x docs (**verified**). This phase uses its model-centric API, `AgentgatewayModel` (`v1alpha1`, on by default):
one resource per model that clients may ask for, attached to the Gateway listener; the gateway builds the routing.
The older route-centric pair `AgentgatewayBackend` + `HTTPRoute` (what the webinar demos use) still works, and
Phase 6 uses it for MCP. v1.7.0 has a built-in `Openrouter` provider (base URL `https://openrouter.ai/api/v1`,
**verified** in `crates/llm/src/custom.rs` at tag v1.7.0).

Read
- https://agentgateway.dev/docs/kubernetes/latest/documentation/quickstart/install/
- https://agentgateway.dev/docs/kubernetes/latest/documentation/quickstart/llm/ (it uses `provider: OpenAI`; you use
  `Openrouter`)
- https://agentgateway.dev/docs/kubernetes/latest/documentation/llm/models/about/ (model-centric vs. route-centric
  table, listener opt-in)
- Old route-centric pattern, for comparison only: `~/Tutorial/webinar-demos/02-multi-agent-skills/gateway/*.yaml`.

Do
- [x] Install Gateway API CRDs v1.6.3 (`kubectl apply --server-side --force-conflicts`), then
      `helm upgrade -i agentgateway-crds` and `agentgateway` v1.7.0 into `agentgateway-system` (**verified**
      commands on the install page). `kubectl api-resources --api-group=agentgateway.dev` must then list
      `AgentgatewayModel`. If `cr.agentgateway.dev` is unreachable from your network, try
      `oci://ghcr.io/agentgateway/charts/...` (the agent-factory demo did this).
- [x] `gateway/gateway.yaml`: Gateway `ai-gateway`, class `agentgateway`, HTTP :80, routes from all namespaces, and
      `allowedRoutes.kinds` listing both `AgentgatewayModel` (group `agentgateway.dev`) and `HTTPRoute`. The model
      kind turns on the LLM paths (`/v1/chat/completions`, `/v1/models`, ...); `HTTPRoute` keeps the listener open
      for Phase 6. In kind the Service stays `<pending>` as LoadBalancer; that is fine for in-cluster use (or set
      ClusterIP via `AgentgatewayParameters`, see `~/Tutorial/agent-factory-demo/platform/gateway/gateway.yaml`).
- [x] Secret `openrouter-secret` in `agentgateway-system` with the **raw** OpenRouter key under the key name
      `Authorization` (no `Bearer`; the gateway adds it, **verified** in the LLM quickstart). Create it from `.env`
      through stdin, the same way you created `kagent-openrouter` in Phase 1.
- [x] `gateway/openrouter-model.yaml`: one `AgentgatewayModel` in `agentgateway-system` with `parentRefs` (the
      Gateway, section `http`), `provider: Openrouter`, `match.model: anthropic/claude-haiku-5.5` and
      `policies.auth.secretRef` (`kubectl explain agentgatewaymodel.spec`). The model name goes in `match.model`
      because `metadata.name` cannot contain `/`. Check that `status.parents` says Accepted, then test through a
      port-forward of the proxy: `GET /v1/models`, then one `POST /v1/chat/completions` with `max_tokens: 16`.
- [x] New ModelConfig `openrouter-via-gateway` in `kagent`: `provider: OpenAI`,
      `openAI.baseUrl: http://ai-gateway.agentgateway-system.svc.cluster.local/v1`, the same model
      `anthropic/claude-haiku-5.5`, and a dummy key Secret (the gateway holds the real one). Point
      `my-first-template` at it, wait for the new revision, new session, invoke. Confirm the request in the gateway
      proxy logs.
- [x] When the new session works, delete the direct ModelConfig `openrouter-haiku-model-config` and the Secret
      `kagent-openrouter`. **Find out**: what happens to an old Session pinned to the revision that used them?
- [x] **Find out**: did the egress gateway let the agent reach the in-cluster gateway without an `EgressPolicy`?
      (Model-provider destinations are inferred, **verified**, but the docs show no in-cluster example.) If not,
      write `agents/50-egress.yaml` with the gateway origin and add `egressPolicyRefs` to the Agent.
      Debug with the `EgressPoliciesApplied` condition and `kubectl logs -n ate-system deploy/atenet-egress`.

Check
- [x] `kubectl get secret -n kagent` shows the dummy key only; the real key exists only in `agentgateway-system`.
      A `hello` turn appears in the gateway logs.
- [x] A request for a model you did not declare (for example a Sonnet ID) is refused by the gateway with
      `model_not_found` and never reaches OpenRouter. That is your first cost control.

Teach-back: draw the request path Session, egress gateway, ai-gateway, OpenRouter, and name which component holds
which credential. Then say in one sentence when you would use `AgentgatewayModel` and when
`AgentgatewayBackend` + `HTTPRoute`.

Pitfalls
- 401: check the Secret name and key (`Authorization`). In v1.7.0 a `Bearer ` prefix in the Secret is stripped
  (**verified** with `kubectl explain agentgatewaymodel.spec.policies.auth`), so it is no longer the usual cause.
- `model_not_found`: the model name kagent sends matches no `match.model`, or the listener does not allow the
  `AgentgatewayModel` kind.
- `AgentgatewayModel` is `v1alpha1`: fields can change in the next release. Trust `kubectl explain` on your cluster
  over blog posts and old demos.
- An `AgentgatewayPolicy` cannot target an `AgentgatewayModel`; per-model policies go in its inline `spec.policies`
  (matters in Phase 6).
- Do not pin an older agentgateway chart (v1.0.x fails the CRD cost budget on Kubernetes 1.37).

---

## Phase 3: Tools: GitHub MCP, approvals, first useful agent (4–6 h)

Goal: give an agent real tools and a human gate before it writes anything.

Read
- https://kagent.dev/docs/kagent/1.x/skills-and-mcp/about-tools/
- https://kagent.dev/docs/kagent/1.x/get-started/your-first-mcp-tool/
- https://kagent.dev/docs/kagent/1.x/agents/human-in-the-loop/
- https://docs.github.com/en/copilot/how-tos/provide-context/use-mcp/set-up-the-github-mcp-server
  (remote server, PAT auth, toolsets)

Do
- [ ] Secret `github-pat` in `kagent` from `.env`. `agents/20-mcp-github.yaml`: `RemoteMCPServer github` with
      `url: https://api.githubcopilot.com/mcp/`, `protocol: STREAMABLE_HTTP`, `headersFrom` `Authorization` from the Secret.
      **Find out**: whether the stored value must be `Bearer <pat>` or the raw token (GitHub docs say
      `Authorization: Bearer`; the webinar stored the raw token). Optionally add header
      `X-MCP-Toolsets: repos,issues,pull_requests` to shrink the tool list.
- [ ] Verify discovery (**verified** commands): `kubectl get remotemcpserver -n kagent` shows ACCEPTED True;
      list `.status.discoveredTools`. Write down the tool names you will use.
- [ ] Agent `repo-scout`: read-only binding (`get_file_contents`, `list_issues`, `search_code`, or whatever
      discovery shows). Ask it to describe the empty `kagent-todo-app` repo and list issues.
- [ ] Add a second binding on the same server with write tools (`create_branch`, `create_or_update_file`,
      `push_files`, `create_pull_request`) and `requireApproval: true`. Ask it to add a `CONTRIBUTING.md` via PR.
      The CLI will stop at "Input required" (**verified**); approve from the UI (port 8082).
      Reject once and read what the model does with the reason.
- [ ] Preview of devops: Agent `cluster-reader` bound to the built-in `kagent-tool-server` with `k8s_get_resources`,
      `k8s_describe_resource`, `k8s_get_pod_logs`, `k8s_get_events`. Ask about pods in `kagent`.
- [ ] Fallback if the hosted GitHub MCP is unreachable from your network: run `ghcr.io/github/github-mcp-server`
      in-cluster (Deployment + Service + RemoteMCPServer with the FQDN URL).
      **Find out**: the flag for streamable-HTTP transport in the current image.

Check
- [ ] One PR opened by an agent after your approval; the PAT appears in exactly one Secret.

Teach-back: what the model sees for a ToolBinding; where approval is enforced (harness, not the MCP server);
why the same server can appear in two bindings.

---

## Phase 4: The team builds the todo app (12–16 h)

Goal: `team-lead`, `backend-dev`, `frontend-dev`, `devops` deliver the app through PRs you review, and `devops`
deploys it into the cluster.

### 4.0 Spec and backlog (you write these, 1 h)
- [ ] In `kagent-todo-app/README.md`: FastAPI app `app/main.py`, SQLite file, endpoints `GET/POST /api/todos`,
      `PATCH /api/todos/{id}`, `DELETE /api/todos/{id}`, `GET /healthz`; static `app/static/index.html` + `app.js`
      served by FastAPI; `tests/test_api.py` smoke test with `pytest`; `Dockerfile`; `k8s/` (Deployment, Service
      NodePort 30080, namespace `todo`); `.github/workflows/build.yml` building to
      `ghcr.io/<github-user>/kagent-todo-app:<sha>`.
      Coding rules: complete files, no diffs; one PR per issue; PR body lists files changed and how to test.
- [ ] Create GitHub issues as the backlog: #1 skeleton + `/healthz` + test; #2 todos API;
      #3 frontend list/add/toggle/delete; #4 Dockerfile + CI image; #5 k8s manifests + deploy;
      #6 due date field end to end; #7 filter done/open. Keep each small.

### 4.1 Skills as OCI images (2 h)
- [ ] `skills/backend-dev/SKILL.md` (front matter `name`, `description`, then rules: FastAPI style, tests, file layout),
      same for `frontend-dev` and `devops`. Dockerfile `FROM scratch` + `COPY SKILL.md /SKILL.md`
      (model on `~/Tutorial/agent-factory-demo/agents/skills/python-coding/Dockerfile`).
- [ ] `docker login ghcr.io` with a PAT that has `write:packages` (a classic PAT for packages is fine; keep it separate
      from the fine-grained repo PAT), build, push to `ghcr.io/<github-user>/kagent-skills/backend-dev`, make the
      package public, copy the **digest**. Skills must be pinned by digest (**verified**); git sources do not become
      ready in rc3 (**verified**).
- [ ] Reference them in the templates:
      `spec.skills: - name: backend-dev, source: {oci: ghcr.io/<github-user>/kagent-skills/backend-dev@sha256:...}`.

### 4.2 Specialists alone (3–4 h)
- [ ] A second `AgentgatewayModel` in `gateway/` for the Sonnet ID, and ModelConfig `sonnet-via-gateway` (same
      gateway `baseUrl`) in `agents/10-modelconfigs.yaml`. Coding agents use it; `hello` and `team-lead` stay on
      haiku until 4.3.
- [ ] `30-backend-dev.yaml`, `31-frontend-dev.yaml`: AgentTemplate (role prompt, repo layout from the README,
      GitHub write tools with `requireApproval` on `create_pull_request` only) + Agent on your Harness.
- [ ] `32-devops.yaml`: GitHub tools + `kagent-tool-server` tools `k8s_apply_manifest`, `k8s_get_resources`,
      `k8s_rollout`, `k8s_get_pod_logs`; `requireApproval` on the binding with `k8s_apply_manifest` and on
      `k8s_delete_resource`.
- [ ] Run issue #1 through `backend-dev` (UI session so you can approve). Review the PR on GitHub, run it locally
      (`uvicorn`, `pytest`), merge. Then #2, then #3 with `frontend-dev`.
- [ ] Count tokens and cost per issue on the OpenRouter activity page; write it in the log.

### 4.3 Team lead, two patterns (3–4 h)
- [ ] **Pattern A, hand-off**: `40-team-lead.yaml` with `tools: - subAgent: {name, description, templateRef}` for the
      three specialists. Send issue #6. Observe (**verified** behaviour): the turn is transferred to one specialist
      and stays there; the lead does not get the result back and cannot sequence backend, frontend, devops.
      Write down when this pattern is enough (routing) and when it is not (orchestration).
- [ ] **Pattern B, call-and-return**: `21-mcp-kagent.yaml`: `RemoteMCPServer kagent-agents` with
      `url: http://kagent-controller.kagent.svc.cluster.local:8083/mcp`, `STREAMABLE_HTTP`. New `team-lead` template
      binding `list_agents`, `invoke_agent`, `get_session`, with a prompt: split the issue into backend, frontend,
      devops tasks; call each specialist in order with the PR link of the previous step; report all PR links.
      Send issue #6 again.
      Limitation to design around (**verified**): a blocking `invoke_agent` cannot surface a specialist's approval
      question. So for specialists called this way, remove `requireApproval` and let **PR review** be the human gate;
      keep `requireApproval` only on `devops` cluster writes, and run devops directly (not via the lead) or tell the
      lead to stop before deploy. **Find out**: does the lead's identity (`X-User-Id`) matter for `invoke_agent` permissions.
- [ ] Compare A and B in the log: which one would you use for a real team, and why.

### 4.4 Deploy (2 h)
- [ ] Issue #4: `devops` writes Dockerfile + GitHub Actions; merge; confirm the image in ghcr.io (public).
- [ ] Issue #5: `devops` writes `k8s/` and applies to namespace `todo` after your approval.
      Open http://localhost:8080 (kind maps 30080). Add a todo. Celebrate.
- [ ] Break it on purpose: scale the Deployment to 0 or set a wrong image tag, ask `devops` to diagnose and fix
      (approval on apply).

Check
- [ ] Issues #1 to #5 closed by agent PRs you reviewed; app runs in the cluster; a log entry with tokens/cost per issue
      and the A-vs-B comparison.

Teach-back: explain to a colleague, without the docs, how a message from you reaches a specialist and comes back
under Pattern B, naming every component it crosses.

Pitfalls
- A new template revision needs a new Session; the UI makes that obvious, the CLI does not.
- Specialists on the kagent engine only have the tools you bound: no shell, no test runs. That is why PR review + CI
  is the gate here.
- Keep tool lists short and prompts explicit about PR body format; it changes quality more than the model does.

---

## Phase 5: agentregistry as the catalog (4–6 h)

Goal: see what a registry adds on top of "YAML in git": discovery, versions, a shared catalog of MCP servers,
skills and agents.

Read
- https://github.com/agentregistry-dev/agentregistry (README, `charts/agentregistry`, release notes for v0.4.0)
- Local: `~/Tutorial/webinar-demos/03-agent-registry/readme.md` and `k8s.md` (0.3.3 workflows),
  `~/Tutorial/webinar-demos/02-multi-agent-skills/scripts/02-install-stack.sh` (Helm flags, pgvector Postgres, NodePort 30121)

Do
- [ ] Install `arctl` (read the script first):
      ```bash
      curl -fsSL https://raw.githubusercontent.com/agentregistry-dev/agentregistry/main/scripts/get-arctl | bash
      ```
- [ ] Install agentregistry v0.4.0 on the cluster with Helm (**find out**: OCI chart URL, or clone the repo and install
      `charts/agentregistry`; v0.4.0 needs a fresh database and `ar.dev/v1alpha1` manifests, **verified** not compatible
      with 0.3.x). Use a pgvector Postgres image, anonymous auth on, NodePort 30121 so http://localhost:12121 works.
- [ ] `registry/` manifests: the GitHub MCP server, your three skills (OCI refs), your four agents.
      `arctl apply -f registry/`. Browse the UI, search from `arctl`.
- [ ] **Find out** and document: can a v0.4.0 `Runtime` deploy an agent to kagent **1.x**? (0.3.3 used the 0.x REST API
      that 1.x removed, **verified**.) If not, the registry is a catalog here; write what would need to exist for it to deploy.
- [ ] Turn anonymous auth off; see what breaks for `arctl`.

Check
- [ ] Search finds your MCP server, skills and agents; you can state in one paragraph what the registry gives you
      that this git repo does not.

---

## Phase 6 (optional): governance and traces through the gateway (4–6 h)

- [ ] Put the GitHub MCP server behind agentgateway (`AgentgatewayBackend` of type MCP + HTTPRoute `/github-mcp`),
      move the PAT into `agentgateway-system`, repoint the `RemoteMCPServer`. Now both credentials live in the gateway.
- [ ] Expose `team-lead` over A2A through the gateway with an `AgentgatewayPolicy` `apiKeyAuthentication`;
      call it with and without the key. (Pattern: `~/Tutorial/webinar-demos/youtube-series/03-virtual-mcp/readme.md`.)
- [ ] Add a rate limit and a prompt guard for the OpenRouter model (inline `spec.policies` on the
      `AgentgatewayModel`, or an `AgentgatewayPolicy` on the Gateway for all models; **find out** which policies the
      inline field supports); trigger each once.
- [ ] Install the lightweight OTel stack (https://kagent.dev/docs/kagent/1.x/observability/lightweight-otel-stack/)
      and read one full team run as a trace.

## Phase 7 (optional): Claude Code harness comparison (3–4 h)

Read: https://kagent.dev/docs/kagent/1.x/agents/agent-harness/ (Claude section)
- [ ] Harness `claude-harness` (`spec.claude: {}`), ModelConfig Anthropic **without** `promptCaching`
      (**verified**: it is rejected), AgentTemplate with the `backend-dev` prompt and the GitHub `RemoteMCPServer`
      (whole server; partial tool selection is ignored on this harness, **verified**).
      No Anthropic key: point `anthropic.baseUrl` at the gateway (its `Openrouter` preset also accepts the Anthropic
      Messages format, **verified** in source); **find out** whether the harness works through it.
- [ ] `EgressPolicy` for `https://github.com` and `https://pypi.org` / `https://files.pythonhosted.org` so Claude Code
      can clone, `pip install`, run `pytest`, push. **Find out**: how to let git push without the PAT entering the sandbox.
      Candidate: git sends a placeholder header via `-c http.extraHeader="Authorization: Basic x"` and the EgressPolicy
      replaces it with `prefix: "Basic "` + a Secret holding `base64(x-access-token:<pat>)`.
- [ ] Run issue #7 with it. Compare with Phase 4: PR quality, whether tests were actually run, tokens, wall time,
      what you lost (per-tool approval, structured output, sub-agent tools).

## Phase 8: Retro (1 h)

`notes/RETRO.md`: for kagent, agentgateway, agentregistry and Agent Substrate, one paragraph each: what it is for,
what it did well, what hurt, open **find out** items you could not answer, and anything worth reporting upstream
(rc3 bugs you hit). Decide what you would build next.

- [ ] Upgrade kagent `v1.0.0-rc3` → `v1.0.0` with Helm (CRDs, Substrate, kagent; `pg_dump` first; see the upgrade
      line in Deep dives). Then retest `invoke_agent` from Claude Code and report it upstream if it still fails.
      The Helm-owned `default-model-config` + Secret `kagent-anthropic` were deleted by hand in Phase 2 (2026-11-12);
      set the kagent values so the upgrade does not recreate them with a real key (**find out** which values:
      `helm get values kagent`, https://kagent.dev/docs/kagent/1.x/reference/helm/).

---

## Open questions list (keep current in `notes/LOG.md`)

- Helm 4 with the agentgateway and agentregistry charts: any failure?
- Exact CLI config key for the default provider (`kagent config schema`).
- Which Substrate object is the running Actor; what a suspended Session looks like.
- Is an EgressPolicy needed for the in-cluster gateway `baseUrl`?
- GitHub MCP header: raw token or `Bearer`?
- Does the lead's identity (`X-User-Id`) affect `invoke_agent` permissions?
- agentregistry v0.4.0 Helm source and kagent 1.x Runtime support.
- Claude harness: credential-free git push via EgressPolicy.

## Deep dives for later (not needed to finish a phase)

The goal of each phase is the main concepts. When a detail comes up, park it here in one line and move on.
Read these after Phase 8, or when one blocks you.

From Phase 1 (2026-11-10):
- Revisions: `desiredRevision` vs `latestSuccessfulRevision` on the Agent, Session `preparedRevision`,
  how a ModelConfig change makes a new revision, when old revisions are garbage-collected.
- Session states: what `RUNTIME_STATE_READY` and `SUSPENDED` really mean; is there an idle timeout?
- Actor lifecycle: checkpoint after every task (`fidelity: volumes`, ~0.1 s), wake-up on the next task (`/readyz`),
  whether snapshots go to RustFS (`s3://ate-snapshots/kagent/`), what survives deleting the `kagent-default` pod.
- Conversation store: SQLite on the Actor's `/data` (rc3 `go/adk/pkg/session/local.go`) vs. the A2A tasks kept
  in the controller's Postgres.
- Egress: model destinations inferred from the ModelConfig, credential injection at the gateway,
  `EgressPolicy` + `spec.egressPolicyRefs` (`docs/architecture/egress-policies.md` in the kagent repo).
- Naming: runtime app name `kagent__NS__my_first_agent`, ActorTemplate `<agent>-<revision[:12]>`, "atespace".
- Identity: `--user-id` (default `admin@kagent.dev`) is a data partition; which identity the UI and Claude Code use.
- CLI connection: the CLI talks to `localhost:8083`, so it needs a port-forward to `svc/kagent-controller`.
- Cluster incident: node left with no Docker network (kind "old IPv4 but no current IPv4"), exit 137 vs 1,
  the `127.0.0.1:12121` clash with the old `multi-agent-demo` cluster, controller/UI restart counts.
- Upgrade path rc3 → 1.0.0 with Helm: CRDs, Substrate, kagent; `pg_dump` first; moving a CLI install to Helm
  (https://kagent.dev/docs/kagent/latest/operations/upgrade/).
- OpenRouter: `provider: OpenAI` + `openAI.baseUrl: https://openrouter.ai/api/v1`, model ID with a dot
  (`anthropic/claude-haiku-5.5`), why the `/v1` path works.
- MCP `invoke_agent` from Claude Code: kagent created the Session and Task, but Claude Code rejected the reply as a
  "malformed task handle" and the Task was canceled after ~1 s. Retest after the 1.0.0 upgrade; if it still fails,
  report it upstream (Phase 8).

## Facts verified on 2026-11-09 (sources)

- kagent releases: https://github.com/kagent-dev/kagent/releases (latest tag v1.0.0-rc3, Nov 6)
- kagent 1.0 release notes: https://kagent.dev/docs/kagent/1.x/reference/release-notes/1.0/
- Installation (Kubernetes 1.37, kind 0.32+, CLI path, Helm path): https://kagent.dev/docs/kagent/1.x/setup/installation/
- Harness types and limits: https://kagent.dev/docs/kagent/1.x/agents/agent-harness/
- Delegation (subAgent hand-off): https://kagent.dev/docs/kagent/1.x/examples/agent-delegation/
- Agents via MCP (`invoke_agent`): https://kagent.dev/docs/kagent/1.x/examples/agents-via-mcp/
- Skills (OCI only works in rc3): https://kagent.dev/docs/kagent/1.x/skills-and-mcp/skills/
- Egress policies: https://kagent.dev/docs/kagent/1.x/agents/egress-policies/
- Human in the loop: https://kagent.dev/docs/kagent/1.x/agents/human-in-the-loop/
- Tools ecosystem (`kagent-tool-server` tool names): https://kagent.dev/docs/kagent/1.x/reference/tools-ecosystem/
- Helm values: https://kagent.dev/docs/kagent/1.x/reference/helm/
- agentgateway releases (v1.7.0): https://github.com/agentgateway/agentgateway/releases
- agentgateway install: https://agentgateway.dev/docs/kubernetes/latest/documentation/quickstart/install/
- agentregistry releases (v0.4.0): https://github.com/agentregistry-dev/agentregistry/releases

## Facts verified on 2026-11-10 (sources)

- kagent `v1.0.0` published 2026-11-09: https://github.com/kagent-dev/kagent/releases/tag/v1.0.0
- agentgateway v1.7.0 (2026-10-27) is still the newest release; the "latest" docs are 1.7.x (older ones under
  1.4.x–1.6.x): https://github.com/agentgateway/agentgateway/releases
- `AgentgatewayModel` (`agentgateway.dev/v1alpha1`) ships in the v1.7.0 `agentgateway-crds` chart; `Openrouter` is
  in its `spec.provider` enum.
- `Openrouter` preset: base URL `https://openrouter.ai/api/v1`; formats Completions, Messages, Responses,
  Embeddings, Rerank: https://github.com/agentgateway/agentgateway/blob/v1.7.0/crates/llm/src/custom.rs
- Model-centric vs. route-centric, listener opt-in, policies on models:
  https://agentgateway.dev/docs/kubernetes/latest/documentation/llm/models/about/
- LLM quickstart (Secret key `Authorization` with the raw key, `allowedRoutes.kinds`, `AgentgatewayModel`):
  https://agentgateway.dev/docs/kubernetes/latest/documentation/quickstart/llm/
