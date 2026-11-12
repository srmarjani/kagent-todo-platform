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

## 2026-11-12 — Phase 3: tools, GitHub MCP, approvals

- Secret `github-pat` (key `Authorization`, value `Bearer <pat>` from `.env` via stdin). kagent sends a header
  exactly as stored, so the `Bearer ` prefix is needed (find-out answered). agentgateway adds it itself; kagent does not.
- `RemoteMCPServer github` (`api.githubcopilot.com/mcp/`, header `X-MCP-Toolsets: repos,issues,pull_requests`):
  discovery from the controller found 39 tools.
- `repo-scout`, 7 read tools: correctly reported one README and 0 issues. 4 model calls (6.4k → 7.5k input tokens),
  10 MCP calls. No EgressPolicy needed for GitHub (inferred from the RemoteMCPServer).
- Mistake: `server.apiGroup` in a tool binding is rejected ("apiGroup must be omitted"). Server dry-run caught it.
- Second binding on the same server: 4 write tools with `requireApproval: true`. In the UI I rejected the first
  `create_branch` with "use branch name docs/contributing"; the model retried with that name. PR #1 (CONTRIBUTING.md)
  opened after my approvals. PR author is me (`srmarjani`): the agent acts as the PAT owner.
- Check: the PAT is in exactly 1 of 31 Secrets (`kagent/github-pat`).
- `cluster-reader`, 4 read-only k8s tools: traced all `kagent` pod restarts to a PostgreSQL restart at 04:15 UTC
  (exit 255, likely node stop). 8 kubectl calls, none on Secrets.

### Concepts
- Three layers: RemoteMCPServer (where the tools are) → binding in the AgentTemplate (which tools this agent gets,
  with or without approval) → the model (sees only name, description and input schema of the bound tools).
- Approval is enforced in the kagent harness inside the Actor: the loop pauses before the call leaves the sandbox.
  A rejection goes back to the model as the tool result; GitHub never sees it.
- One server in two bindings = free reads, gated writes. Tools in no binding (merge, delete) cannot even be requested.
- Tools cost tokens on every call: ~4k extra input tokens for 7 GitHub tools. Each agent-loop round resends the
  whole conversation, so input grows with every tool result.
- A binding limits what the model sees, not what the server can do: `kagent-tool-server` is cluster-admin and
  could read Secrets. Real limits need RBAC (Deep dives).

### Open
- Agent identity on GitHub (bot account / GitHub App) and at the AI gateway.
- Read-only RBAC for `kagent-tool-server`.
- Merge PR #1 (my call).

## 2026-11-12 — Phase 4.0–4.2: spec, skills, specialists, issues #2–#4

- 4.0: README spec (stack, API table, agent rules) pushed to kagent-todo-app `master`; backlog issues #2–#8
  (PR #1 took number 1). My spec's API table already had `?status=` and date validation, so #3/#4 also did parts
  of #7/#8. Lesson: spec and backlog must not overlap.
- 4.1: three skills (`SKILL.md` + `FROM scratch` Dockerfile) pushed to `ghcr.io/srmarjani/kagent-skills/*`,
  made public, pinned by digest. Used `gh auth refresh -s write:packages` instead of a separate classic PAT.
  First push of a new GHCR package can fail once with "denied"; a retry works.
- 4.2: Sonnet 5.5 as a second `AgentgatewayModel` (20× Haiku's price). `backend-dev`, `frontend-dev`, `devops`:
  Sonnet + skill + GitHub (read free, `create_branch` free, `push_files`/`create_or_update_file`/`create_pull_request`
  gated); `devops` also k8s read free, `k8s_apply_manifest` gated, no delete/rollout at all.
- The model opens its skill on demand: `load_skill` shows up as a tool call right at the start.

| Issue | Agent | PR | Model calls | Cost | Result |
|---|---|---|---|---|---|
| #2 skeleton | backend-dev | #9 | 6 | ≈ $0.18 | pytest 1 passed, /healthz 200 |
| #3 todos API | backend-dev | #10 | 7 | ≈ $0.32 | pytest 15 passed, curl checks match spec |
| #4 frontend | frontend-dev | #11 | 12 + 3 hung | ≈ $0.36 | 8/8 manual browser steps pass |

Costs from the `ai-gateway` log (Sonnet $2/M in, $10/M out). No prompt caching (`cache_read=0`).

### Incident: frontend-dev hangs
- Three times between ~10:25 and 10:42 a streaming Sonnet call got no first token (up to 8 min 21 s). kagent and
  the gateway just waited; nothing logged until the request ended.
- Tests: Sonnet non-streaming OK, Sonnet streaming 503 "connection timed out" once then OK, Haiku OK, direct to
  OpenRouter OK. Intermittent upstream problem, not my setup. After a 503 the agent retried by itself.
- How to see it: tool call completed + no new `protocol=llm` line at the gateway = stuck, not thinking.
- agentgateway `timeouts`/`retry` only apply to models attached via an HTTPRoute, not to models attached directly
  to the Gateway. A reason for the route-centric style later.

### Findings
- Agents pin old library versions (FastAPI 0.115.6): a model pins what it remembers.
- Frontend race: typing while a POST is in flight gets wiped by `form.reset()`. Small; future issue.
- I merged PR #11 before testing it. Better: test first, then merge.

### Open
- 4.3 team lead; issues #5–#8.
- Gateway timeout + retry (Phase 6), prompt caching (Phase 6).

## 2026-11-12 — Phase 4.3 Pattern A, and the kagent 1.0.0 upgrade

### Pattern A: hand-off team lead (`team-lead-handoff`)
- `subAgent` bindings point at the specialists' templates; kagent compiles the lead + specialists into ONE
  Actor in the lead's Session. To the lead's model a sub-agent is just a tool; the hand-off is one tool call,
  `transfer_to_agent`, and the specialist keeps the conversation.
- `subAgent.agentRef` (a sub-agent with its own Session) is deferred in 1.0 (delegation page note). Pattern B
  builds that by hand via the controller's MCP tools instead.
- Issue #7: the lead routed it to `backend_dev`, which did BOTH halves (API validation + UI overdue marking).
  Routing works; splitting work across specialists does not. PR #12 (opened by hand, see below): 40 tests pass,
  overdue marking correct in the browser.

### rc3 blocker → upgrade
- rc3 connects every MCP binding when an Actor starts or resumes; the Substrate wake-up probe gives up after 30 s.
  The lead tree had 12 bindings (≈36 s): "input was not accepted; retry after the session becomes available".
- Trimmed to 5 bindings (merged read + create_branch per specialist, dropped devops from the lead): start 11 s.
  But resume after an approval once took 20 s for ONE GitHub connection → probe missed again → approval loop.
- Fix = kagent 1.0.0 (#3560, MCP discovery deferred to first use). After the upgrade the Actor is ready at once.

### Upgrade rc3 → 1.0.0 (Helm)
- Backup first: `pg_dump` inside the Postgres pod (no password), 41 tables, kagent + substrate schemas.
- Substrate was already 1.0.0 with the `GoldenEgressPolicy` gate: no Substrate step needed.
- `helm upgrade kagent-crds` → `helm upgrade kagent -f kagent/kagent-values-1.0.0.yaml`. The 1.0.0 controller
  ran DB migration 2 → 3 on start. All 7 Agents got NEW revisions (the new controller compiles differently) and
  were Ready ~3 min later, after golden snapshots were rebuilt.
- The Anthropic key was still in the Helm release values history (`providers.anthropic.apiKey`). New values:
  default model = gateway + dummy key, no `apiKey` (chart creates no Secret then). Deleted the old release
  revision Secret; a scan of all 31 Secrets now finds the Anthropic key nowhere, the OpenRouter key only in
  the gateway, the PAT only in `kagent/github-pat`.
- `--history-max 1` still keeps the previous revision; delete it by hand if it holds secrets.
- CLI: `get-kagent --version v1.0.0` needs sudo; the password prompt timed out once.

### New on 1.0.0: GitHub MCP shows a PR form
- kagent 1.0.0 advertises MCP Apps; GitHub's server then answers `create_pull_request` with an interactive form
  for the user instead of creating the PR. The model retried, I approved again: a loop. Opened PR #12 by hand.

### Open
- Make agents create PRs again on 1.0.0 (switch off MCP Apps or the GitHub form). Needed before Pattern B.
- Pattern B (call-and-return via `invoke_agent`), issues #5, #6, #8. Retest `invoke_agent` from Claude Code.

## 2026-11-12 — Phase 4.3 Pattern B (call-and-return) and the A-vs-B comparison

- GitHub PR form fix: header `X-MCP-Features: mcp_apps_disable_form_deferral` on the `github` RemoteMCPServer
  (GitHub's server defers write tools to an MCP App form when the client supports UI; kagent 1.0.0 does).
  Changing a shared MCP server gave new revisions to exactly the 5 agents that bind it.
- Issue #8 closed (already built by #10/#11). New issue #13 "Clear completed todos" (backend then frontend).
- `team-lead-orchestrator` (Haiku) with the controller's own MCP (`kagent-api`: list_agents, invoke_agent,
  get_session, get_task). It started backend-dev, then frontend-dev, each in its own Session; stopped while I
  approved in the specialist's Session; got PR #14 (backend, "Part of #13") and PR #15 (frontend, stacked on
  the backend branch, "Closes #13"). Header fix confirmed: the agents created the PRs.
- Tested: 48 tests pass, DELETE without/other status → 422, browser button clears only done todos.
- Cost: lead 12 Haiku calls $0.01, specialists 17 Sonnet calls $0.83 → $0.84.

### Findings
- `invoke_agent` works from inside kagent on 1.0.0 (the rc3 "malformed task handle" was the Claude Code client).
- `invoke_agent` blocks until the new Session's Actor accepts the turn (>30 s); `kagent-api` has `timeout: 30s`,
  so the first calls timed out. Retries reused the same `message_id` → still only ONE Session per specialist.
- No blocking "wait": `get_session` says "poll with 1 until the last turn is final". The "stop and tell the
  user" rule kept polling to 3 calls.
- My prompt said "small max_bytes"; the tool needs ≥ 4096. The lead corrected itself after one error.
- Stacked PRs: merging #15 before #14's branch was deleted merged it into the backend branch, not master.
  Fixed with PR #16. Order: merge base PR → delete its branch (GitHub retargets) → merge the next.

### A vs B — which for a real team?
| | A: hand-off (`subAgent`) | B: call-and-return (`invoke_agent`) |
|---|---|---|
| Runs in | one Actor, one Session | one Session per specialist |
| Lead after delegating | gone (specialist keeps the chat) | keeps control, reads results |
| Multi-step issues | one specialist does everything | backend → frontend, stacked PRs |
| Approvals | in the same chat | in each specialist's Session |
| rc3 | broke (MCP bindings at startup) | fine on 1.0.0 |
| Use it for | routing a request to the right expert | orchestrating work that has an order |
Real team: B for issues that span roles; A only as a front-door router.

## 2026-11-12 — Phase 4.4: CI, deploy, break-and-fix

- #5 (devops, direct chat): Dockerfile + GitHub Actions in PR #17. The PAT already had Workflows permission.
  Local test: image builds (208 MB), container serves /healthz, UI and POST. First CI run passed in 48 s and
  pushed `:594cf43…` and `:latest`. The package was public at once: a package pushed by a workflow with
  GITHUB_TOKEN is linked to its repo and inherits the repo's visibility (the skills, pushed from my laptop,
  started private).
- #6 (devops): `k8s/` in PR #18 (namespace todo, Deployment on the exact SHA, readiness /healthz, DB on emptyDir,
  NodePort 30080). Validated with server dry-run (namespace) + strict schema check, nodePort free. After merge
  devops applied 3 manifests (each approved by me); the tool server log shows exactly 3 applies + 9 reads, all in
  `-n todo`. Image pull 1 m 17 s. http://localhost:8080 serves the app; a todo added in the browser was stored.
- Break-and-fix: I set `strategy: Recreate` + a typo image tag → ImagePullBackOff, app down. devops found it
  from events, applied the git manifest (one approval), checked its own fix. App back.
- Drift lesson: `kubectl apply` only manages fields the manifest lists. The git manifest has no `strategy`, so
  `Recreate` survived and `kubectl diff` shows nothing. Write every field you care about into the manifest.

| Task | Agent | PR | Model calls | Cost |
|---|---|---|---|---|
| #5 Dockerfile + CI | devops | #17 | 7 | ≈ $0.24 |
| #6 k8s + deploy | devops | #18 | 17 | ≈ $0.72 |
| Fix the outage | devops | — | 8 | ≈ $0.47 |

Phase 4 total for the agents ≈ $3.5 (Sonnet for specialists; Haiku for leads was cents).

### Findings
- GitHub annotations: Node.js 20 actions are forced to Node 24, and `ubuntu-latest` moves to Ubuntu 26.
  A small future issue: bump action versions.
- Container runs as root; DB on emptyDir (data lost on restart). Future issues: non-root user, PVC, explicit
  `strategy`, resources and a liveness probe.
- The fix cost more than the feature (36k input tokens per call at the end): every investigation result
  (events JSON, describe, logs) stays in the conversation and is resent each round.

### Open
- Teach-back answer (Pattern B round trip).
- Phase 5 (agentregistry), Phase 6 (gateway timeouts, prompt caching, ACL), Phase 7, Phase 8.

## 2026-11-12 — Phase 5: agentregistry as the catalog

- agentregistry v0.4.0 is still the newest release. Chart via OCI
  (`oci://ghcr.io/agentregistry-dev/agentregistry/charts/agentregistry --version 0.4.0`), namespace
  `agentregistry`, bundled Postgres 18 on a 5 Gi PVC (no pgvector needed), NodePort 30121 → http://localhost:12121.
  `arctl` v0.4.0 from the release asset, checksum verified, in `~/.local/bin`.
- `registry/` (ar.dev/v1alpha1): 3 remote MCP servers (github with placeholder token, kagent-tools, kagent-agents),
  3 skills (source = git at commit 9507b4c; the registry has no OCI source for skills, so the digest is in the
  description), 4 agents (backenddev, frontenddev, devops, teamlead) referencing skills and MCP servers.
- Gotchas: agent names can't contain hyphens (they become Python identifiers); agents with skills need
  `compatibleHarnesses` (`{type: kagent}`); `--dry-run` can't resolve references to entries in the same run;
  `arctl get` defaults to tag `latest`, so `--tag v1` is needed; the CLI has no free-text search, the UI does
  (and matches descriptions).
- Deploy to kagent 1.x: no. The kagent runtime adapter (even on main) calls kagent 0.x REST (`/api/agents`,
  `/api/toolservers`); kagent 1.x answers every unknown path with an empty 200, so it would not even fail loudly.
  Here the registry is a catalog only.
- Auth: nothing to turn off. No authn configured, `PublicAuthzProvider` allows every action, tokens are ignored,
  anonymous writes work. Same pattern as kagent: hooks shipped, allow-all default. OIDC is open issue #712.

### Check: what the registry gives that this git repo does not
Git holds the files; the registry holds a searchable catalog of what exists and how it fits together. Anyone can
find "which MCP servers and skills can I use, and which agents already use them" in a web UI with search over
names and descriptions, without knowing this repo, YAML layout or kubectl. Entries are versioned with tags
(`v1`, `stable`) so teams can promote and pin versions side by side, and references between agents, skills and
MCP servers are resolved and checked on publish (a dangling reference is rejected). It can also expose an
MCP-registry-compatible API (off by default) for AI clients. What it does not give here: deployment to
kagent 1.x, and any access control; so today it is a read-mostly catalog that should be published from reviewed
git by CI.

## 2026-11-12 — Phase 6.1: GitHub MCP behind the gateway

- `gateway/github-mcp.yaml`: `AgentgatewayBackend github-mcp` (`spec.mcp`, static target
  `api.githubcopilot.com:443/mcp/`, StreamableHTTP, `policies.tls.sni`, `policies.auth.secretRef`) plus
  `HTTPRoute /github-mcp` on `ai-gateway`, which sets `X-MCP-Toolsets` and `X-MCP-Features`.
- Secret `github-mcp-secret` (key `Authorization`, raw PAT, from `.env` via stdin); the gateway adds `Bearer `.
  The docs put the PAT inline in the HTTPRoute; not needed, `auth.secretRef` works on an MCP target.
- Headers set by an HTTPRoute filter reach the MCP target: `tools/list` through the gateway gave 39 tools,
  the same as direct.
- `RemoteMCPServer github` now has only the URL `http://ai-gateway.agentgateway-system.svc.cluster.local/github-mcp`.
  6 agents rebuilt (~3 min). No EgressPolicy: egress inferred from the URL, as for the ModelConfig in Phase 2.
- Gateway log: one line per MCP call with `mcp.method.name`, `mcp.target`, `gen_ai.tool.name` (e.g. `list_issues`).
  Discovery comes from `kagent-controller`, the Actor's calls from `atenet-egress`: still no agent identity.
- Real MCP proxy: the client's session ID differs from the upstream one (long encoded ID; maybe signed with
  `ai-gateway-session-key`, not verified).
- Deleted `kagent/github-pat`: the PAT is in 1 of 33 Secrets. Old Sessions on earlier revisions lose GitHub.

### Open
- No client auth on the `ai-gateway` listener: any pod that reaches it can use the PAT and the OpenRouter key.
- `registry/10-mcp-servers.yaml` still says the PAT is in `kagent/github-pat`.
