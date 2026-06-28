# Replicating Claude "dynamic workflows" in Omnigent — feasibility investigation

Status: investigation / proposal (no code changes)
Author: investigation spike
Question asked: *Is it possible to replicate Claude Code's "dynamic workflows" in
Omnigent?*

**Short answer: yes — and more cheaply than first expected.** The fan-out of
parallel agents already exists. The defining property — moving *the plan itself
into code*, a deterministic orchestration script that holds the loop and
intermediate results outside any LLM context — initially looks net-new, but
Omnigent already ships a **full HTTP/SSE API and a typed Python client SDK
(`omnigent_client`)**. So the orchestration script is a *usage pattern over
existing infrastructure*, not a new runtime: a deterministic Python program can
create sub-agent sessions, fan turns out to them in parallel, hold every result
in its own variables, and return only the final answer. See §2.5 — this is
exactly the design the user proposed, and it's the recommended path.

---

## 1. What Claude dynamic workflows actually are

Source: <https://code.claude.com/docs/en/workflows> and
<https://claude.com/blog/introducing-dynamic-workflows-in-claude-code>.

A *dynamic workflow* is **a JavaScript script that Claude writes** and a
background runtime executes, separate from the conversation. The script
orchestrates [subagents](https://code.claude.com/docs/en/sub-agents) at scale.

Key properties (from the docs):

- **The plan lives in code, not in a context window.** The script holds the
  loop, the branching, and every intermediate result in script variables.
  Claude's context only ever sees the final answer. This is the defining
  difference from subagents / skills / agent teams, where Claude is the
  orchestrator and decides turn-by-turn what to spawn.
- **Scale:** dozens to hundreds of agents per run. Hard caps: **≤16 concurrent
  agents** (fewer on low-core machines), **1,000 agents total per run**.
- **Quality patterns, not just more agents.** Because the orchestration is code,
  it can codify adversarial cross-review (independent agents review each other's
  findings before they're reported) or multi-angle drafting + weighing.
- **Background + resumable.** Runs in the background while the session stays
  responsive; resumable *within the same session* (completed agents return cached
  results, the rest run live).
- **Constraints the runtime enforces:** no mid-run user input (only agent
  permission prompts can pause it); the script itself has **no direct filesystem
  or shell access** — only the agents read/write/run; agents always run in
  `acceptEdits` and inherit the user's tool allowlist.
- **Invocation:** the `ultracode` keyword in a prompt, `/effort ultracode` (auto
  for every substantive task), or a saved/bundled command (e.g. `/deep-research`).
  Saved scripts live in `.claude/workflows/` or `~/.claude/workflows/` and become
  `/<name>` commands; they read invocation input from a global `args`.
- **The script is a real artifact:** written to a file under the session dir, so
  the user can read it, diff it across runs, edit it, and re-launch.

The docs' own comparison table is the cleanest framing of where this sits:

| | Subagents | Skills | Agent teams | Workflows |
|---|---|---|---|---|
| What it is | A worker Claude spawns | Instructions Claude follows | A lead agent supervising peers | **A script the runtime executes** |
| Who decides what runs next | Claude, turn by turn | Claude | The lead agent, turn by turn | **The script** |
| Where intermediate results live | Context window | Context window | A shared task list | **Script variables** |
| Scale | A few per turn | Same | A handful | **Dozens–hundreds per run** |

---

## 2. What Omnigent has today

Omnigent is a meta-harness. Its multi-agent story maps almost exactly onto the
**"agent teams"** column above: an **LLM orchestrator decides turn-by-turn** what
to spawn, and intermediate state lives in its context (plus, in Polly's case, a
`.polly/registry.json` task list on disk).

The substrate that a workflow runtime would build on is already there:

- **Asynchronous, parallel sub-agent dispatch.** `sys_session_send`
  (`omnigent/tools/builtins/spawn.py`) launches a sub-agent as an independent
  task and returns a non-blocking handle
  (`{task_id, kind: "sub_agent", conversation_id, status, …}`, the
  `_AsyncToolHandle` shape from `omnigent/runtime/workflow.py`). Results
  auto-deliver over the unified `async_work_complete` topic; the orchestrator
  collects them via the inbox (`sys_read_inbox`) rather than busy-polling. So
  *fan-out and join already exist* — they are just driven by an LLM turn loop
  today.
- **Programmatic session creation.** `spawn: true` registers
  `sys_session_create`, letting an agent launch an existing agent by id *or
  author a custom agent config and launch it via `config_path`* — i.e. agents
  can mint new agents at runtime.
- **Mixed harnesses / models per worker.** Each sub-agent picks its own
  `executor.harness` + `model` (`docs/AGENT_YAML_SPEC.md`). A workflow could
  route the cheap wide fan-out to a small model and the final review to a strong
  one — Polly already does this per dispatch via `args.model`.
- **Cross-vendor adversarial review is a worked example.** Polly
  (`examples/polly/config.yaml`) already implements "implementer's diff reviewed
  by a *different vendor*" — the exact quality pattern dynamic workflows
  advertise — but as prompt instructions an LLM follows, not as code.
- **Fan-out bounds already exist as policy.** Polly caps fan-out with the
  `spawn_bounds` policy (`max_dispatches_per_turn: 5`,
  `dispatch_tools: [sys_session_send, sys_session_create]`). This is the natural
  hook for the workflow caps (16 concurrent / 1000 total).
- **Crash-durable agent loop.** `omnigent/runtime/workflow.py` is the core agent
  loop, "all durably checkpointed for crash recovery" — relevant to the
  resumability requirement.
- **Parent/child wake + block escalation.** `subagent_block_notifier.py` already
  handles the messy edge case dynamic workflows call out (an agent blocked on a
  permission prompt mid-run) by escalating to the parent.

The one thing Omnigent does **not** ship as a feature: a way to express the
orchestration as **deterministic code that runs outside an LLM context**. Today
every "orchestrator" is itself an LLM agent (Polly's brain is a `claude-sdk`
agent). There is no `.omnigent/workflows/` script concept, no `ultracode`-style
trigger. (Note: the `ucode` / `UcodeAgentState` symbols in the codebase are
unrelated — that is Databricks workspace config, not Claude's `ultracode`.)

## 2.5. The decisive enabler: Omnigent has a full HTTP/SSE API *and* a client SDK

This reshapes the whole feasibility picture. Omnigent is not just a CLI — it is a
**server with a documented REST + SSE API** (`openapi.json`, ~10k lines) and a
**typed Python client SDK** (`sdks/python-client/omnigent_client`). That means
the "plan-as-code outside the LLM context" primitive does **not** require a
bespoke new runtime — *it already exists as a programmable surface a deterministic
script can drive.*

The API surface relevant to orchestration (from `openapi.json`):

- `POST /v1/sessions` — create a session (the unit a sub-agent runs in). NB: the
  public create takes an agent bundle + metadata (title/labels/effort/workspace);
  it does **not** expose a `parent_session_id`. Parenting a session into the
  orchestrator's tree is done today by the *internal* `sys_session_send` /
  `sys_session_create` tools, not this endpoint — see the confidence note in §7.
- `GET /v1/sessions/{id}/child_sessions` (read the sub-agent tree; the SDK's
  `child_sessions_tree` / `subtree_busy` are built "for an SDK driver") and
  `POST /v1/sessions/{source}/fork` (deep-copy a session's history into a new
  one).
- `POST` to a session + `GET /v1/sessions/{id}/stream` (SSE) and
  `GET /v1/sessions/{id}/items` — drive a turn and read results.
- `.../resources/files`, `.../resources/environments/{id}/{filesystem,shell,search}`,
  `.../resources/terminals` — read files, run shell, search inside a session's
  environment.
- `.../agent`, `.../switch-agent`, `.../policies`, `.../permissions` — manage the
  agent, its model/harness, and its governance per session.

The Python SDK (`omnigent_client`) wraps all of this with typed ergonomics:

```python
from omnigent_client import OmnigentClient, BlockStream, pipe, skip_intermediate_ends

async with OmnigentClient(base_url=SERVER, headers={"Authorization": f"Bearer {TOK}"}) as c:
    session = c.session(model="some-agent")        # create / drive a session
    result = await c.query(model="agent", input="…")  # one-shot, returns .text / .files
    async for block in pipe(BlockStream().stream(session, "…"), skip_intermediate_ends()):
        ...                                         # stream typed blocks
```

It supports session create, multi-turn `send`/`query`, SSE streaming as raw
events *or* semantic blocks, client-side tool handling, and `fork`. There is also
a `LocalServer` helper for spinning a server up in-process.

**So the user's hypothesis is correct and is the cleanest route to a faithful
replica:** an orchestrator can author a **deterministic Python program** that
imports `omnigent_client` (or hits the REST API directly), and that program — not
an LLM turn loop — creates the sub-agent sessions, fans out turns to them in
parallel (`asyncio.gather` over many `session.send`/`query` calls), holds every
intermediate result **in its own variables**, applies branching / cross-review /
synthesis in plain code, and surfaces only the final answer. That *is* Claude's
"plan-as-code, state-in-script-variables, runtime executes it" model — built on
shipping infrastructure rather than a new engine.

### The one piece of real plumbing to verify

A program the orchestrator launches (via its `sys_os_shell` / a terminal, or as a
standalone process on the host) needs two things to reach the server:

1. **A base URL** for the Omnigent server it belongs to, and
2. **An auth token / credential** for it.

The host/runner the agent runs on is *already authenticated* to the server
(that's what `omnigent login` / `omnigent host` establish, with credentials under
`~/.omnigent`), so the credential exists on the box. The open question is whether
it is conveniently exposed to the agent's child process (an env var like a
`OMNIGENT_BASE_URL` + token, or a readable credential file). If not already
threaded through, exposing a scoped session token to the agent's environment is a
**small, well-contained plumbing task** — and the natural thing to add to make
this pattern first-class. The `OmnigentClient` already accepts `headers=` and an
`httpx.Auth` for exactly this.

---

## 3. Gap analysis

| Capability | Claude workflows | Omnigent today | Gap |
|---|---|---|---|
| Parallel sub-agent fan-out / join | ✅ runtime | ✅ `sys_session_send` + inbox | none |
| Mixed harness/model per worker | ✅ | ✅ | none |
| Adversarial cross-review pattern | ✅ codified in script | ⚠️ prompt-driven (Polly) | codify in code |
| Concurrency / total caps | ✅ 16 / 1000 | ⚠️ per-turn policy only | extend policy |
| **Plan-as-code (state in variables, not context)** | ✅ | ⚠️ no *feature*, but the HTTP API + `omnigent_client` SDK make it scriptable today | wrap, don't build |
| **Background runtime executing a script** | ✅ | ⚠️ run the script as a process/task on the host | small |
| **Resumable run (cached completed agents)** | ✅ | ⚠️ checkpointed loop, no run-graph cache | extend |
| Saved workflow → `/command` with `args` | ✅ | ⚠️ skills/agents, no script command | net-new (thin) |
| `ultracode` trigger / auto-plan | ✅ | ❌ | optional |
| Script artifact on disk (readable/editable) | ✅ | ⚠️ a `.py` orchestration script is exactly this | trivial |

The crucial correction vs a first reading: the "net-new runtime" rows are **not**
net-new. Because Omnigent exposes a full HTTP/SSE API *and* a Python client SDK
(§2.5), the script and its runtime are a **usage pattern over existing
infrastructure**, not a new engine to build.

---

## 4. Feasibility verdict

**Replicable: yes.** Two viable strategies, from cheapest to most faithful.

### Approach A — "Workflow agent" (no new runtime; ~days)
Ship a bundled orchestrator agent (a Polly variant) whose *skill* is to author a
plan and fan it out, plus a lightweight Python "plan file" that the agent reads.
This gets the *behavior* (scale, cross-review, one final answer) without the
defining property (plan-in-code-outside-context). State still lives in the
orchestrator LLM's context, so it does not scale to hundreds of agents the way a
real workflow does, and "resumable from cached results" is only as good as the
checkpointed loop. **Good as a fast proof-of-concept; not a true replica.**

### Approach B — script-driven workflow over the existing API/SDK (faithful; the recommended replica)
This is the honest replica, and §2.5 shows it does **not** require a bespoke
runtime — it reuses the HTTP API + `omnigent_client`. The orchestrator authors a
**deterministic Python script** and runs it; the script holds the plan and state.

Sketch, reusing existing pieces:

1. **Workflow artifact.** A `.py` script in `.omnigent/workflows/<name>` plus an
   invocation `args` (read as a global / argv). Python is the direct analog to
   Claude's JS scripts; Omnigent is Python and `omnigent/tools/_pep723.py`
   already runs PEP-723 scripts with inline deps. The script *is* the
   readable/editable/diffable artifact Claude advertises.
2. **The orchestration SDK already exists: `omnigent_client`.** The script does
   not need a new API — it creates sessions, sends turns, streams results, and
   forks via the SDK. A thin convenience wrapper can expose workflow-shaped
   helpers over it:
   - `spawn(agent, prompt, *, model=None)` → `client.session(...).send(...)`
   - `gather(...)` → `asyncio.gather` over many sends (parallel fan-out/join)
   - `review(diff, contract, *, by="different-vendor")` → spawn a reviewer agent
   Intermediate results live in the script's variables — exactly the property
   that keeps them out of any LLM context.
3. **Execution + credentials.** Run the script as a host process/task the
   orchestrator launches; thread a scoped server base-URL + token into its
   environment (the one real plumbing item, §2.5). The conversation receives only
   the final return value. For background + resumability, run it as a
   session-scoped task and reuse the durable-checkpoint pattern from
   `runtime/workflow.py`.
4. **Caps + governance.** Enforce ≤16 concurrent / 1000 total by generalizing the
   `spawn_bounds` policy from per-turn to per-run, gated through the existing
   Nessie policy layer so server/agent/session policies still apply. Because
   session creation goes through the same API, server-side policies already see
   every spawned agent.
5. **Invocation + management UI.** A trigger (a `/workflow` command or an
   `ultracode`-style keyword) and a progress view. The spawned sessions are
   ordinary child sessions, so they already appear in the web UI Subagents panel
   and session tree — a `/workflows` progress view is mostly a projection over
   existing sub-agent state, not new transport.
6. **Save / reuse.** A finished run's script saved into `.omnigent/workflows/`
   becomes a re-runnable command — directly analogous to Claude's save flow and
   compatible with Omnigent's existing skill/agent bundle distribution.

**Distinct Omnigent advantage:** because Omnigent is multi-harness, a workflow
here could fan out across *Claude Code, Codex, Cursor, Pi, …* in one run —
something Claude's own workflows (single vendor) cannot do. The cross-vendor
review Polly already performs becomes a first-class, codified workflow step.

---

## 5. Risks / open questions

- **Script trust & sandboxing.** A Python orchestration script is code the agent
  wrote; it must run with the FS/shell denied to the script itself (only agents
  act), exactly as Claude constrains it. The existing sandbox + policy layers
  cover the agents, but the *script host* needs its own confinement. A
  declarative phase DSL sidesteps this entirely at the cost of expressiveness.
- **Cost blow-up.** Hundreds of agents per run is real spend. Reuse the existing
  cost policies (`cost_budget`) and surface per-agent token usage as Claude does.
- **Resumability semantics.** "Cached completed agents" needs a run-graph keyed
  store; the checkpointed loop is a foundation but not the whole thing.
- **Concurrency on a single host.** 16 concurrent harness subprocesses is heavy;
  Omnigent's managed-host / cloud-sandbox story could actually *exceed* Claude's
  local cap by distributing agents across sandboxes.

---

## 6. Recommendation

The key finding (§2.5): Omnigent already exposes a full HTTP/SSE API **and** a
typed Python client SDK, so "plan-as-code outside the LLM context" — the one
property that defines dynamic workflows — is achievable by having the orchestrator
write a deterministic Python program that drives `omnigent_client`. It is a usage
pattern over shipping infrastructure, not a new runtime.

1. **Now (proof of concept, hours-to-days):** prove the loop end-to-end with a
   standalone Python script using `omnigent_client` that creates N agent
   sessions, fans out with `asyncio.gather`, does cross-vendor review in code, and
   prints one synthesized answer. This validates §2.5 with zero core changes and
   surfaces the two open items (credential threading, and whether the spawned
   sessions need to be parented via the internal `sys_session_*` tools to appear
   in the UI tree).
2. **Then (faithful replica, Approach B):** make it first-class — a
   `.omnigent/workflows/<name>.py` artifact, a thin `spawn`/`gather`/`review`
   wrapper over `omnigent_client`, scoped server credentials threaded into the
   orchestrator's environment, per-run `spawn_bounds` caps, background execution,
   and a saved-workflow `/command`. An `ultracode`-style auto-trigger is optional
   polish on top.
3. **Approach A** (a Polly-fork orchestrator agent that codifies fan-out +
   cross-review purely in prompts) remains a fine *no-code-change* demo of the
   behavior, but it keeps state in the LLM context, so it is the fallback, not the
   target.

The fan-out/join substrate, mixed-model workers, cross-vendor review, fan-out
caps, a durable loop, **a REST API, and a client SDK** are all already in the
tree. The genuinely new work shrinks to: a scoped credential for the script,
a thin workflow wrapper + `/command` surface, and per-run caps.

---

## 7. Confidence and what remains unverified

This investigation is **static** — based on reading code and docs on this branch.
**Nothing here was executed** (no live server, no run script). That is the single
biggest caveat.

**High confidence (read directly):**
- The HTTP/SSE API and the `omnigent_client` SDK exist and expose session create,
  send (`post_event`), SSE `stream`, `get`, `list_items`, `interrupt`, `compact`,
  `fork`, model override, elicitation resolve, and a `child_sessions` tree reader
  explicitly described as "the queryable rollup an SDK driver needs"
  (`sdks/python-client/omnigent_client/_sessions.py`).
- A deterministic program can drive multiple sessions in parallel
  (`asyncio.gather`) and hold all intermediate state in its own variables — the
  defining property of dynamic workflows. *This core thesis is the high-confidence
  part.*

**Medium / low confidence (inference or contradicted by a closer read):**
- **Parented sub-agents via the public API.** The public `sessions.create()` has
  **no `parent_session_id`** — it takes an agent bundle + metadata. So a script
  can create and drive N sessions, but making them appear as nested children in
  the orchestrator's UI tree is *not* confirmed via the public API; today that is
  the job of the internal `sys_session_send` / `sys_session_create` tools. This
  may need those tools or a small API addition.
- **Credential exposure.** Whether a process the agent launches automatically
  receives the server base URL + a usable token is unverified (the host is
  authenticated; convenient exposure to the child process is the open item).

**Calibrated estimate:** ~85% that the core thesis holds (deterministic
program + existing API/SDK ⇒ parallel agents with state outside the LLM context,
no new runtime); ~50% on the convenience details (parented tree via public API,
low-effort credentials). A small executed proof-of-concept against a local
server would move both to high confidence and is the recommended next step.

### Sources
- Orchestrate subagents at scale with dynamic workflows — <https://code.claude.com/docs/en/workflows>
- Introducing dynamic workflows in Claude Code — <https://claude.com/blog/introducing-dynamic-workflows-in-claude-code>
- A harness for every task: dynamic workflows in Claude Code — <https://claude.com/blog/a-harness-for-every-task-dynamic-workflows-in-claude-code>
- Omnigent in-tree references: `openapi.json` (REST/SSE API), `sdks/python-client/omnigent_client/` + `sdks/README.md` (Python client SDK), `omnigent/server/routes/sessions.py`, `omnigent/tools/builtins/spawn.py`, `omnigent/runtime/workflow.py`, `omnigent/runtime/subagent_block_notifier.py`, `examples/polly/config.yaml`, `docs/AGENT_YAML_SPEC.md`
