# Replicating Claude "dynamic workflows" in Omnigent — feasibility investigation

Status: investigation / proposal (no code changes)
Author: investigation spike
Question asked: *Is it possible to replicate Claude Code's "dynamic workflows" in
Omnigent?*

**Short answer: yes, and most of the runtime substrate already exists.** The
hard part is not the fan-out of parallel agents (Omnigent already does that); it
is moving *the plan itself into code* — a deterministic, resumable orchestration
script that holds the loop and the intermediate results outside any LLM context.
That is a net-new primitive, but it composes cleanly with what Omnigent has.

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

The one thing Omnigent does **not** have: a way to express the orchestration as
**deterministic code that runs outside an LLM context**. Today every
"orchestrator" is itself an LLM agent (Polly's brain is a `claude-sdk` agent).
There is no `.omnigent/workflows/` script concept, no non-LLM runtime that holds
the plan, and no `ultracode`-style trigger. (Note: the `ucode` /
`UcodeAgentState` symbols in the codebase are unrelated — that is Databricks
workspace config, not Claude's `ultracode`.)

---

## 3. Gap analysis

| Capability | Claude workflows | Omnigent today | Gap |
|---|---|---|---|
| Parallel sub-agent fan-out / join | ✅ runtime | ✅ `sys_session_send` + inbox | none |
| Mixed harness/model per worker | ✅ | ✅ | none |
| Adversarial cross-review pattern | ✅ codified in script | ⚠️ prompt-driven (Polly) | codify in code |
| Concurrency / total caps | ✅ 16 / 1000 | ⚠️ per-turn policy only | extend policy |
| **Plan-as-code (state in variables, not context)** | ✅ | ❌ | **net-new** |
| **Background runtime executing a script** | ✅ | ❌ (loop runs an LLM) | **net-new** |
| **Resumable run (cached completed agents)** | ✅ | ⚠️ checkpointed loop, no run-graph cache | extend |
| Saved workflow → `/command` with `args` | ✅ | ⚠️ skills/agents, no script command | net-new |
| `ultracode` trigger / auto-plan | ✅ | ❌ | optional |
| Script artifact on disk (readable/editable) | ✅ | ❌ | net-new |

So ~half the matrix is already met; the workflow-specific half is the work.

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

### Approach B — true workflow runtime (faithful; larger)
Add a first-class, non-LLM orchestration runtime. This is the honest replica.

Sketch, reusing existing pieces:

1. **Workflow spec / artifact.** A script in `.omnigent/workflows/<name>` plus an
   invocation `args`. To avoid embedding a JS engine, make the script **Python**
   (Omnigent is Python; `omnigent/tools/_pep723.py` already runs PEP-723
   scripts) or a small declarative phase DSL (YAML: phases → fan-out spec →
   join → review). Python is the more direct analog to Claude's JS scripts.
2. **A workflow SDK exposed to the script** — the script must *not* touch the FS
   or shell directly (mirror Claude's constraint); it only orchestrates agents:
   - `spawn(agent, prompt, *, model=None) -> handle`
   - `gather(handles) -> results` (await many)
   - `review(diff, contract, *, by="different-vendor") -> verdict`
   These are thin wrappers over the existing `sys_session_send` / inbox /
   `sys_session_create` machinery — the plumbing is done; this is an API surface.
3. **A runtime that executes the script in the background**, holding intermediate
   results in script variables (the whole point). Run it as a session-scoped task
   so it survives alongside the conversation; the conversation only receives the
   final result. Reuse the durable-checkpoint pattern from
   `runtime/workflow.py` for resume.
4. **Caps + governance.** Enforce ≤16 concurrent / 1000 total by generalizing the
   `spawn_bounds` policy from per-turn to per-run, gated through the existing
   Nessie policy layer so server/agent/session policies still apply.
5. **Invocation + management UI.** A trigger (a `/workflow` command or an
   `ultracode`-style keyword) and a progress view. Omnigent already streams
   sub-agent activity to the web UI Subagents panel and the session tree, so a
   `/workflows`-style progress view is mostly a projection over existing
   sub-agent state, not new transport.
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

1. **Now:** ship **Approach A** as a bundled "workflow" orchestrator agent +
   skill (a Polly fork that codifies fan-out + cross-review and returns a single
   synthesized answer). Low risk, demonstrates the user-visible behavior, reuses
   100% existing primitives.
2. **Next:** build **Approach B** — a real workflow runtime with a Python (or
   DSL) script, a small orchestration SDK over the existing sub-agent plumbing,
   per-run caps, background execution, and a saved-workflow `/command`. This is
   the faithful replica and plays to Omnigent's multi-harness strength.

The fan-out/join substrate, mixed-model workers, cross-vendor review, fan-out
caps, and a durable loop are all already in the tree. The genuinely new work is
*plan-as-code executed outside the LLM context* and its management surface.

---

### Sources
- Orchestrate subagents at scale with dynamic workflows — <https://code.claude.com/docs/en/workflows>
- Introducing dynamic workflows in Claude Code — <https://claude.com/blog/introducing-dynamic-workflows-in-claude-code>
- A harness for every task: dynamic workflows in Claude Code — <https://claude.com/blog/a-harness-for-every-task-dynamic-workflows-in-claude-code>
- Omnigent in-tree references: `omnigent/tools/builtins/spawn.py`, `omnigent/runtime/workflow.py`, `omnigent/runtime/subagent_block_notifier.py`, `examples/polly/config.yaml`, `docs/AGENT_YAML_SPEC.md`
