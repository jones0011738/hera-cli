# Hera — Grok-Build parity features

Hera folds the genuine architectural differentiators of xAI's open-source
[Grok Build](https://github.com/xai-org/grok-build) into the single-file Python
CLI (`cli/hera.py`), with **no new dependencies**. Four capabilities:

1. Bounded parallel sub-agent pool
2. Task-graph plan mode
3. ACP (Agent Client Protocol) front-end
4. Fullscreen TUI

Everything routes through the same auth-proxy (`:8090`) as the rest of the stack,
so the context-window guarantee still applies.

---

## 1. Parallel sub-agent pool

When the model delegates several independent subtasks at once, they run through a
**bounded pool** instead of spawning unbounded threads.

- At most `HERA_SUBAGENT_POOL` sub-agents run concurrently (**default 8**); the
  rest queue and start as slots free. Results stay in the order requested.
- `task(inherit_context=true)` hands each sub-agent a compact slice of the parent
  conversation (the overall goal + latest progress) so fan-out agents understand
  the bigger picture.

```
# the model calls, conceptually:
task(tasks=[
  {"description": "audit auth.py for injection"},
  {"description": "audit db.py for injection"},
  {"description": "audit api.py for injection"},
], inherit_context=true)
```

Env: `HERA_SUBAGENT_POOL=8`.

---

## 2. Task-graph plan mode

For a larger job whose parts have dependencies, the model declares a **DAG** and
executes it — independent branches run in parallel, dependents wait and receive
their prerequisites' results.

- `plan_graph(nodes=[...])` — each node is `{id, title, detail?, deps:[ids]}`.
  Validated for duplicate ids, unknown/self deps, and cycles (must be a DAG).
- `run_plan_graph(inherit_context=true)` — dispatches every node whose deps are
  done as a parallel **wave** through the pool above, forwarding prerequisite
  results forward. A node whose prerequisite failed is skipped (the skip cascades
  down that branch only). Returns a per-node report to synthesize.

```
plan_graph(nodes=[
  {"id":"1","title":"Map the config readers"},
  {"id":"2","title":"Refactor to a central loader","deps":["1"]},
  {"id":"3","title":"Audit the tests","deps":["1"]},
  {"id":"4","title":"Update docs","deps":["2","3"]},
])
run_plan_graph()
# wave 1: [1]  →  wave 2: [2,3] in parallel  →  wave 3: [4]
```

Use `todo_write` for a simple linear checklist; `plan_graph` for a real
dependency graph. In `--tui` the graph renders live in a sidebar (see §4).

---

## 3. ACP (Agent Client Protocol) — `hera --acp`

A second **headless front-end** speaking Zed's
[Agent Client Protocol](https://agentclientprotocol.com): newline-delimited
**JSON-RPC 2.0 over stdio**. This lets ACP editors/orchestrators drive Hera, and
lets Hera run alongside other ACP agents in one pipeline. The bespoke `--serve`
protocol (used by the VS Code extension) is unchanged.

Methods handled (client → agent): `initialize`, `authenticate`, `session/new`,
`session/load`, `session/prompt`, `session/cancel`.

Notifications/requests (agent → client):
- `session/update` — `agent_message_chunk`, `agent_thought_chunk`,
  `tool_call` / `tool_call_update` (with `diff` content), `plan`.
- `session/request_permission` — tool/plan/install approvals.
- `fs/write_text_file` / `fs/read_text_file` — **editor-native file apply/read**,
  used only when the client advertises the matching `fs` capability in
  `initialize`; otherwise Hera writes through its own tools.

`session/load` replays a saved conversation's history back to the client as
`session/update` chunks before responding.

Quick check:

```
printf '%s\n' \
  '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":1,"clientCapabilities":{}}}' \
  '{"jsonrpc":"2.0","id":2,"method":"session/new","params":{"cwd":".","mcpServers":[]}}' \
  | hera --acp
```

---

## 4. Fullscreen TUI — `hera --tui`

A fullscreen `curses` UI (stdlib, no dependency) with:

- a **scrollback pane** (mouse wheel / PgUp-PgDn to scroll),
- an **input line** with basic editing,
- a **status bar** (model · tokens · auto-mode · activity),
- a **live task-graph sidebar** that updates as `plan_graph`/`run_plan_graph`
  run — **click a node** to open a popup with that node's full sub-agent result.

Approvals appear as modal prompts (`[y]/[a]/[n]` for tools, `[1]/[2]/[3]` for
plans). `Ctrl-C` interrupts a running turn; `Esc` / `Ctrl-D` (or `/exit`) quits.
Slash commands available in the TUI: `/clear`, `/undo`, `/resume [n]`.

It **falls back to the plain REPL** when curses isn't available (non-TTY,
`TERM=dumb`, Windows console) or on any curses error, so `--tui` is always safe
to pass.

---

## Where it lives

| Feature | Entry points in `cli/hera.py` | Tests |
|---|---|---|
| Sub-agent pool | `run_subagents_parallel`, `_run_subagent_wave`, `run_subagent(inherit=)`, `_inherit_slice` | `tests/test_subagent_pool.py` |
| Task graph | `tool_plan_graph`, `run_plan_graph`, `_render_plan_graph_text` | `tests/test_plan_graph.py` |
| ACP | `acp_main`, `AcpPeer`, `_acp_emit`, `_acp_editor_apply` | `tests/test_acp.py` |
| TUI | `tui_main`, `_tui_run`, `TuiModel` | `tests/test_tui.py` |
