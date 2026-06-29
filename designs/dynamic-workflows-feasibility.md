# Replicating Claude "dynamic workflows" in Omnigent — design

Status: **recommended design** (supersedes the earlier feasibility investigation
on this branch; that investigation's findings are folded in as evidence below)
Author: investigation + design spike
Question asked: *Can we replicate Claude Code's "dynamic workflows" in Omnigent,
as securely as possible?*

## Summary

**Yes — and the secure design is clearer than the first investigation suggested.**

The heart of the feature is: an **orchestrator agent decides on a complex
workflow, writes a deterministic program that implements it, and runs that program
without the agent having to "think through" each step**. The program — not an LLM
turn loop — creates sub-agent sessions, fans them out in parallel, holds every
intermediate result in its own variables, and returns only the final answer. The
sub-agents it spawns appear as children of the orchestrator's session.

The one design decision that makes this both faithful *and* secure is **where the
program runs**: not as a sandboxed *agent payload* (a `sys_os_shell` child, which
is deliberately walled off from server credentials — see §5), but as a
**trusted, runner-side orchestration task** confined to a single capability:
talking to the local Omnigent session API with a **scoped, short-lived token**, and
nothing else (no filesystem, no shell, no arbitrary network). That confinement is a
direct mirror of how Claude constrains its own workflow script — "the script itself
has no direct filesystem or shell access; only the agents read/write/run" — and it
is what keeps agent-authored orchestration code from becoming a privilege
escalation.

Everything else the feature needs already ships: the Python client SDK
(`omnigent_client`), parallel sub-agent fan-out/join, mixed-vendor workers,
cross-vendor adversarial review, a crash-durable agent loop, server-side policy and
caps, and a web-UI sub-agent tree. The genuinely new work is two small primitives
(a scoped session token, a parent-aware create) plus thin glue.

---

## 1. What Claude dynamic workflows are

Sources: <https://code.claude.com/docs/en/workflows>,
<https://claude.com/blog/introducing-dynamic-workflows-in-claude-code>.

A *dynamic workflow* is **a script Claude writes** and a background runtime
executes, separate from the conversation. The script orchestrates
[subagents](https://code.claude.com/docs/en/sub-agents) at scale. The defining
properties:

- **The plan lives in code, not in a context window.** The script holds the loop,
  the branching, and every intermediate result in script variables. Claude's
  context only ever sees the final answer. This is the difference from subagents /
  skills / agent teams, where an LLM is the orchestrator and decides turn-by-turn
  what to spawn.
- **Scale:** dozens to hundreds of agents per run. Hard caps: **≤16 concurrent
  agents**, **1,000 agents total per run**.
- **Quality patterns codified in code:** e.g. adversarial cross-review (independent
  agents review each other before findings are reported), multi-angle drafting.
- **Background + resumable** within the same session (completed agents return
  cached results; the rest run live).
- **The script has no direct filesystem or shell access** — only the agents
  read/write/run. Agents run in `acceptEdits` and inherit the user's allowlist.
  *(This constraint is load-bearing for our security model — see §3.2.)*
- **Invocation:** a keyword/effort trigger, or a saved command (`/<name>`) that
  reads its input from a global `args`. The script is a real artifact on disk —
  readable, diffable, editable, re-launchable.

The docs' comparison table is the cleanest framing of where this sits:

| | Subagents | Skills | Agent teams | Workflows |
|---|---|---|---|---|
| What it is | A worker Claude spawns | Instructions Claude follows | A lead agent supervising peers | **A script the runtime executes** |
| Who decides what runs next | Claude, turn by turn | Claude | The lead agent, turn by turn | **The script** |
| Where intermediate results live | Context window | Context window | A shared task list | **Script variables** |
| Scale | A few per turn | Same | A handful | **Dozens–hundreds per run** |

Omnigent's multi-agent story today maps onto the **"agent teams"** column: an LLM
orchestrator decides turn-by-turn what to spawn, and state lives in its context.
The target is the **Workflows** column.

---

## 2. The substrate Omnigent already has

Almost everything a workflow runtime needs is in the tree; only "plan-as-code
outside an LLM context" is missing as a *feature*.

- **A typed Python client SDK — `omnigent_client`.** Omnigent is a server with a
  documented REST + SSE API (`openapi.json`) and a typed client SDK
  (`sdks/python-client/omnigent_client`). The SDK does session create, multi-turn
  `send`/`query`, SSE streaming as raw events or semantic blocks, client-side tool
  handling, `fork`, model override, and a `child_sessions` tree reader described as
  "the queryable rollup an SDK driver needs" (`_sessions.py:684`, `fork` at `:887`).
  *This is the API the workflow program calls.*
- **Parallel sub-agent fan-out / join.** `sys_session_send`
  (`omnigent/tools/builtins/spawn.py`) launches a sub-agent as an independent task
  and returns a non-blocking handle; results auto-deliver over the
  `async_work_complete` topic and are collected via the inbox. Fan-out and join
  already exist — they are just LLM-driven today.
- **Programmatic, parented session creation (internally).** The internal
  `sys_session_create` accepts `parent_conversation_id` and can create an **idle,
  create-only** child (`spawn.py:540`, `:897`) — the exact "provision but don't
  drive" split the design relies on.
- **Mixed harness/model per worker** (`docs/AGENT_YAML_SPEC.md`); Polly already
  routes per dispatch via `args.model`.
- **Cross-vendor adversarial review is a worked example.** Polly
  (`examples/polly/config.yaml`) implements "implementer's diff reviewed by a
  *different vendor*" — but as prompt instructions, not code.
- **Fan-out caps as policy.** Polly's `spawn_bounds`
  (`max_dispatches_per_turn: 5`) is the natural hook for the 16/1000 caps.
- **Crash-durable agent loop** (`omnigent/runtime/workflow.py`, "all durably
  checkpointed for crash recovery") — the foundation for background + resumable.
- **Sub-agent tree + web UI for free.** The tree is *derived* from two stored
  fields (`kind == "sub_agent"`, `parent_conversation_id == <orchestrator id>`);
  `GET /v1/sessions/{id}/child_sessions` is a query over them
  (`sqlalchemy_store.py:820`). Any session carrying the orchestrator's id as parent
  appears in the Subagents panel with no extra wiring.

The one thing absent as a feature: a way to express the orchestration as
**deterministic code that runs outside an LLM context**. Today every orchestrator
is itself an LLM agent. That gap is what §3 fills.

---

## 3. Recommended architecture

### 3.1 Where the program runs — and why it's the crux

Run the workflow program as a **trusted, runner-side orchestration task**, *not* as
a sandboxed agent payload.

The instinct from the first investigation was "the agent launches the script via
`sys_os_shell` and we thread a token into its environment." §5 shows that path is
deliberately walled off: the agent's `sys_os_shell` environment is deny-by-default,
the runner's binding token is *always* stripped, and the on-disk credential file is
masked by the sandbox. That wall is correct and should stay — it exists precisely to
stop agent-authored payloads from exfiltrating credentials.

So invert it. The program is *orchestration*, not *agent work*. Run it on the
trusted side of that wall — the same side as `runtime/workflow.py`'s durable loop
and `_make_auth_token_factory()` — where holding a server credential is legitimate.
This single choice resolves **both** previously-open problems at once:

- **Credentials** stop being a hack: trusted-side code is *meant* to hold a server
  token (§5).
- **Parenting** stops needing a broker: trusted-side code can create parented
  children directly (§3.3), so no LLM round-trip is required to mint them.

### 3.2 The security model (mirrors Claude; arguably tighter)

Trusted-side does **not** mean unconstrained — the program is still
**agent-authored code**, so it runs in a tightly scoped confinement that mirrors
Claude's "the script has no FS/shell; only agents act":

1. **One capability, nothing else.** The program may talk to the **local Omnigent
   session API and nothing else** — no filesystem, no shell, no arbitrary network
   egress. Reuse the existing bwrap/seatbelt sandbox machinery with a dedicated
   *orchestration profile*: network allowlisted to the server only, FS/shell denied.
2. **A scoped, short-lived token — never the user's full bearer.** The program's
   credential is scoped to exactly: *create and drive child sessions owned by this
   user, parented under this one session, subject to the run caps.* It is **not**
   `_make_auth_token_factory()`'s raw OIDC bearer. Inject it via the existing
   `credential_proxy` path (§5).
3. **Server-side policy still applies to every spawn.** Because each create flows
   through the same authorized API, Nessie policies, cost budgets, and the
   per-run caps (§4) all keep applying to every agent the program spawns.

The blast radius of a malicious or buggy workflow program is therefore bounded to
"spawn sub-sessions under my own parent, up to the caps" — no privilege escalation,
no credential theft, no host access. This is **stricter** than Claude's model in one
respect: the agent-authored orchestration code never holds a general-purpose
credential at all, only a parent-scoped one.

> Honest residual risk: we are still *executing agent-authored code*. That is true
> of Claude's workflows too. The isolation that bounds it is the OS sandbox
> (network/FS/shell) **plus** the scoped token — not language-level sandboxing
> (RestrictedPython is fragile; do not rely on it).

### 3.3 Parenting — the tree falls out for free

Children appear under the orchestrator iff they carry its id as
`parent_conversation_id` with `kind="sub_agent"`; the tree is just a query over
those fields (`sqlalchemy_store.py:174`, `:820`). Two verified facts shape the
mechanism (§6): `parent_conversation_id` is set **only at create** and is
immutable, and the **public** create API has no parent parameter — only the
internal `sys_session_create` does.

So the design adds a **parent-aware create** the scoped token may call (the earlier
doc's §7 "Option 2"), wiring straight to the store's existing
`create_conversation(parent_conversation_id=…)` argument. The server enforces that
the created session's parent equals the token's scoped parent — scope and
capability line up exactly, which is what makes this clean. Once the field is
writable through that path, the spawned sessions appear in the Subagents panel with
no further work.

### 3.4 End-to-end flow

1. **Decide + author.** The orchestrator (LLM) decides a workflow is warranted,
   writes a deterministic `workflow.py` against a thin `omnigent_workflow` helper
   (over `omnigent_client`), and triggers it. *This is the only LLM judgment in the
   loop — what to decompose and, later, whether the result is good.*
2. **Launch (trusted-side, sandboxed).** The runner starts `workflow.py` as a
   session-scoped durable task under the §3.2 orchestration profile, injecting the
   scoped parent-bound token.
3. **Run (deterministic).** The program creates parented child sessions, fans out
   with `asyncio.gather`, holds all intermediate state in its own variables,
   applies branching / cross-vendor review / synthesis in plain code.
4. **Children show up live.** They are ordinary parented sub-agents, so the web UI
   Subagents panel and session tree render the run with no new transport.
5. **Return one answer.** Only the final value is posted back to the orchestrator's
   conversation; the intermediate fan-out never enters any LLM context.
6. **Save / reuse.** The `workflow.py` artifact saved to `.omnigent/workflows/<name>`
   becomes a re-runnable `/command` reading its input from `args` — directly
   analogous to Claude's save flow and compatible with bundle distribution.

**Distinct Omnigent advantage:** because Omnigent is multi-harness, one run can fan
out across *Claude Code, Codex, Cursor, Pi, …* — something Claude's single-vendor
workflows cannot. The cross-vendor review Polly does in prompts becomes a
first-class, codified workflow step.

---

## 4. What's new vs. reused

**Build — two primitives + glue:**

| Item | Notes |
|---|---|
| **Scoped session token** | Minted trusted-side; scope = `{user, parent_session_id, caps}`. The security keystone (§3.2). |
| **Parent-aware create** | New write path honoring the existing `(parent_conversation_id, title)` uniqueness + parent-exists checks; server enforces `parent == token.scope.parent` (§3.3). |
| Orchestration sandbox profile | A bwrap/seatbelt profile: net→server only, FS/shell denied. Reuses existing sandbox + `credential_proxy`. |
| `omnigent_workflow` wrapper | Thin `spawn` / `gather` / `review` / `synthesize` over `omnigent_client`. |
| `/workflow` trigger + launch-as-task | Reuse the durable-task pattern from `runtime/workflow.py`. |
| Per-**run** caps | Generalize `spawn_bounds` from per-turn to per-run (16 concurrent / 1000 total), gated server-side. |

**Reuse — already in the tree:** `omnigent_client` SDK; fan-out/join; mixed-vendor
workers; cross-vendor review (Polly); the sandbox + `credential_proxy`; the durable
loop; server-side policy/cost; the web-UI sub-agent tree.

---

## 5. Evidence: the credential boundary (why "trusted-side" is mandatory)

This section records the investigation that drove §3.1/§3.2. It answers: *does a
process the agent launches automatically get a usable server base-URL + token?*

**A usable token is withheld from agent payloads — deliberately, by three locks:**

1. **Deny-by-default env for `sys_os_shell`.** `build_helper_env`
   (`inner/os_env.py:159`) passes through only `_DEFAULT_ENV_PASSTHROUGH`
   (`PATH`, `HOME`, locale, `TERM`, the `OMNIGENT` session *marker*) — no
   credentials. The rationale is stated in-code: otherwise "the helper would just
   call `sys_os_shell('env')` to enumerate every secret and `curl` it out."
2. **The runner auth secret is *always* stripped** — both branches, even
   `sandbox.type: none`: `strip_runner_auth_secrets` removes
   `RUNNER_AUTH_SECRET_ENV_VARS = {RUNNER_TUNNEL_BINDING_TOKEN_ENV_VAR}`
   (`runner/identity.py:58`). Comment: "the helper runs the agent's tool payload,
   which must never see the tunnel binding token."
3. **The on-disk credential is masked.** The user's OIDC bearer lives in
   `~/.omnigent/auth_tokens.json`; the sandbox tmpfs-masks dot-directories
   (`~/.aws`, `~/.ssh`, `~/.config/gcloud`, and by the same rule `~/.omnigent`) —
   `bwrap_sandbox.py:951`. So a sandboxed payload can't read it either.

**But the machinery to do it correctly already exists — for trusted-side code:**

- **A token minter:** `_make_auth_token_factory()` (`runner/_entry.py:271`) mints
  fresh server bearers (stored OIDC from `omnigent login`, or Databricks OAuth).
- **A child-process threading precedent:** the policy callback stamps exactly
  `{server_url, session_id, Bearer token}` into a child process today —
  `OMNIGENT_POLICY_URL` / `OMNIGENT_SESSION_ID` / `OMNIGENT_POLICY_AUTH="Bearer …"`
  (`runner/app.py:1135-1145`), and executors export `_OMNIGENT_SERVER_URL` /
  `_OMNIGENT_SESSION_ID` to harness subprocesses (`cursor_executor.py:413`).
- **A sanctioned per-spec credential path:** `credential_proxy` / `env_passthrough`
  gives a confined subprocess *specific* credentials without exposing the parent's
  full environment.

**Conclusion that drove the design:** the first investigation's "expose a token to
the agent's shell — small plumbing" is the wrong frame. The mechanism is small, but
the easy path is deliberately walled off, and the only readily-available token is
the user's *full* bearer. The honest, secure answer is to run the program
**trusted-side** (where the token factory legitimately lives) and hand it a
**scoped** token via `credential_proxy` — exactly §3.

---

## 6. Parenting mechanics (verified facts)

The store derives the sub-agent tree from `(kind, parent_conversation_id)` and sets
`kind = "sub_agent" if parent_conversation_id else "default"` at create
(`sqlalchemy_store.py:174`). Two facts shape §3.3:

- **`parent_conversation_id` is immutable after create.** `update_conversation`
  (`sqlalchemy_store.py:1780`) accepts only `title, reasoning_effort,
  model_override, cost_control_mode_override, harness_override,
  terminal_launch_args, archived` — **no parent field**, and there is no
  `reparent`. So "adopt an existing top-level session by id" is impossible without a
  core change.
- **The public create API has no parent parameter** — `SessionsNamespace.create`
  (`_sessions.py:338`) takes `bundle, filename, title, labels, reasoning_effort,
  workspace`. Only the internal `sys_session_create` passes
  `parent_conversation_id` through to the store's existing
  `create_conversation(parent_conversation_id=…)` argument.

The parent-aware create in §3.3 is the minimal honest addition: expose that
existing store argument on a write path the scoped token may call, reusing the
parent-exists and `(parent_conversation_id, title)` uniqueness checks already
enforced at create.

---

## 7. Phasing

1. **PoC — prove the loop, zero security work (hours–days).** A standalone Python
   script using `omnigent_client` against a local server: create N sessions (flat /
   top-level), `asyncio.gather` fan-out, cross-vendor review in code, print one
   synthesized answer. Validates the SDK thesis end-to-end. (Uses the full bearer
   and skips parenting — fine for a throwaway local proof.)
2. **Secure target — the §3 architecture.** Scoped token + parent-aware create +
   orchestration sandbox profile + `/workflow` command + per-run caps + durable
   background execution.

---

## 8. Risks / open questions

- **Executing agent-authored code.** Bounded by the OS sandbox (net/FS/shell) +
  scoped token, not by language sandboxing. Same risk class as Claude's workflows.
- **Cost blow-up.** Hundreds of agents is real spend; reuse `cost_budget` and
  surface per-agent usage.
- **Resumability semantics.** "Cached completed agents" needs a run-graph-keyed
  store; the durable checkpointed loop is the foundation but not the whole thing.
  *Note:* a script running as a host process does **not** automatically inherit the
  loop's checkpoints — running it as a session-scoped durable task (§3.4 step 2) is
  what makes background + resume real.
- **Concurrency on one host.** 16 concurrent harness subprocesses is heavy;
  Omnigent's managed-host / cloud-sandbox story could distribute agents across
  sandboxes and *exceed* Claude's local cap.
- **To verify in the PoC.** (a) owner-token `post_event` to a same-owner child is
  permitted (the events route is ownership-gated, not parent-gated — read but not
  executed); (b) the scoped-token mint + `credential_proxy` injection path end to
  end.

---

## 9. What we discarded, and why nothing essential is lost

The earlier investigation explored several shapes. The recommended design absorbs
the good ones and drops two:

- **Approach B (script over the SDK)** — *kept as the foundation.* §3 is Approach B
  with the two vague parts (where it runs, how it authenticates) pinned down.
- **Parent-aware create (old §7 "Option 2")** — *adopted* as §3.3.
- **Flat / top-level sessions (old §7 "Option 3")** — *kept* as the §7 PoC only.
- **Broker pattern (old §7 "Option 1") — discarded.** It had the orchestrator LLM
  mint each child on the program's behalf, putting a nondeterministic LLM turn on
  the program's hot path — the exact latency/nondeterminism dynamic workflows exist
  to remove. The scoped token (§3.2) lets the program tie its own shoes, so the
  broker is unnecessary. Nothing is lost: it was only ever defensible as a
  zero-core-change interim.
- **Re-parenting an existing session by id — discarded as infeasible.**
  `parent_conversation_id` is immutable (§6); adoption would need a core change we
  don't need, since the program creates children already parented.
- **Approach A (prompt-only Polly fork) — demoted to a fallback demo.** It keeps
  state in the LLM context, so it does not scale to hundreds of agents and is not a
  faithful replica. Useful only as a no-code-change behavioral demo.

---

## 10. Confidence

This design rests on a **static** read of code on this branch; **nothing was
executed**. That is the main caveat.

**High confidence (read directly):**
- The SDK exposes session create, `send`/`query`, SSE `stream`, `child_sessions`
  tree, `fork`, model override (`omnigent_client/_sessions.py`).
- A deterministic program can drive many sessions in parallel and hold all state in
  its own variables — the defining property. *The core thesis is the
  high-confidence part.*
- The credential boundary is real and deliberate (§5): agent payloads are denied
  server tokens; a token minter and a child-process threading precedent exist on
  the trusted side. *This is what makes "run it trusted-side" the right call with
  high confidence, where the first investigation only guessed.*
- Parenting is a query over `(kind, parent_conversation_id)`, the field is
  immutable post-create, and only the internal create path sets it (§6).

**Medium / lower confidence (needs the PoC):**
- The scoped-token mint + `credential_proxy` injection wired end to end.
- Parent-aware create exposed on a token-callable write path with `parent == scope`
  enforcement (design is clear; not built).
- Owner-token `post_event` to a same-owner child (ownership-gated route, read but
  not executed).

**Calibrated estimate:** ~90% the core thesis holds (deterministic program +
existing SDK ⇒ parallel agents with state outside the LLM context, no new runtime);
~75% that the secure shape lands close to §3 as described (the boundary evidence in
§5 is the part that moved up from the earlier ~50%). A small executed PoC against a
local server is the recommended next step and would lift both.

### Sources

- Orchestrate subagents at scale with dynamic workflows — <https://code.claude.com/docs/en/workflows>
- Introducing dynamic workflows in Claude Code — <https://claude.com/blog/introducing-dynamic-workflows-in-claude-code>
- A harness for every task: dynamic workflows in Claude Code — <https://claude.com/blog/a-harness-for-every-task-dynamic-workflows-in-claude-code>
- Omnigent in-tree references: `openapi.json`; `sdks/python-client/omnigent_client/` (`_sessions.py`); `omnigent/tools/builtins/spawn.py`; `omnigent/runtime/workflow.py`; `omnigent/inner/os_env.py`; `omnigent/runner/identity.py`; `omnigent/runner/_entry.py`; `omnigent/runner/app.py`; `omnigent/inner/bwrap_sandbox.py`; `omnigent/stores/conversation_store/sqlalchemy_store.py`; `examples/polly/config.yaml`; `docs/AGENT_YAML_SPEC.md`
