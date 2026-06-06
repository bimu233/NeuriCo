# Browser UI for Interactive Mode — Our Changes

This document records **our changes only** — the work to add a browser-based
interface to NeuriCo's interactive mode. Use it as raw material for the PR
description.

> **Baseline:** interactive mode (the LLM-driven manager that runs agents and
> stops to ask the human) already existed as a **terminal-only** feature before
> this work. That baseline is not ours; we build on top of it. Everything below
> is what *we* added or modified.

## TL;DR

We turn the existing **terminal-based** interactive manager into a
**browser-based** experience:

1. A pluggable `UserChannel` abstraction decouples the manager's agent loop from
   the I/O medium (terminal vs. web).
2. An embedded web server gives a real-time **chat with the manager** plus a
   **live agent transcript** in one page.
3. A standalone log **visualizer** that the web server reuses for transcript
   formatting.
4. The manager's system prompt is tuned to interrupt the human far less often.

Web is now the **default** interface; `--cli` falls back to the original
terminal behavior. Autonomous mode (`./neurico run`) is untouched.

## New files (ours)

| File | Purpose |
|---|---|
| `src/interactive/channel.py` | `UserChannel` abstraction. The manager talks to the human via `send()` / `prompt()` / `poll_input()` instead of bare `print`/`input`. Two implementations: `TerminalChannel` (preserves the original behavior) and `WebChannel` (pub/sub queues + SSE fan-out, inbound queue fed by `POST /input`, history replay for reconnects). |
| `src/interactive/web_server.py` | Embedded HTTP server (`InteractiveWebServer`) running in a background thread inside the manager process. Serves one page unifying the **manager chat** (input box, clickable option buttons) and the **live agent transcript** tailed from `workspaces/<ws>/logs/*.jsonl`. Routes: `GET /`, `GET /stream` (Server-Sent Events), `POST /input`. Auto-retries ports if the requested one is taken. |
| `visualizer/visualizer.py` | Standalone read-only log viewer for any workspace (`python visualizer/visualizer.py <workspace>`). The web server imports and reuses its `format_entry` / `TRANSCRIPT_FILES` so the agent feed looks identical in both. |
| `ideas/examples/titanic_survival_prediction.yaml` | Small example idea for testing interactive mode. |

## Modified files (ours)

### `src/interactive/manager.py`
- Threads a `channel` through `InteractiveManager` and `ToolExecutor`.
- Replaces every `print()` / `input()` in the agent loop with
  `channel.send()` / `channel.prompt()` / `channel.status()`.
- Browser interjection: the human can type at any time while an agent runs
  (via `channel.poll_input(timeout=...)`), not just Ctrl+C.
- New CLI flags: `--cli` (terminal instead of browser), `--port N`
  (web port, default 7890), `--no-browser` (don't auto-open the browser).
- Web is the default: builds a `WebChannel` + `InteractiveWebServer`, prints
  the URL, auto-opens the browser, and tears the server down in a `finally`.
- **Re-resolve `idea_file` after the submitted→in_progress move.** Startup looks
  up the idea (capturing its `submitted/` path), then calls
  `IdeaManager.update_status(..., "in_progress")`, which *moves* the file. The
  captured path was passed to every agent unchanged, so the agent opened a
  `submitted/<id>.yaml` that no longer existed (`FileNotFoundError` inside
  `agent_runner.py`). Now `find_idea` is re-run after the move so agents receive
  the file's current `in_progress/` location.

### `src/interactive/tools.py`
- `ToolExecutor` takes a `channel` (defaults to `TerminalChannel` so it still
  works standalone).
- The `ask_user` tool routes through `channel.prompt(message, options)`.
- Coerces `options` when the CLI backend's XML tool-call shim hands it back as
  a JSON-encoded string, so the browser still renders clickable buttons.
- **Translates host paths to in-container paths in `_run_agent`.** The manager
  runs on the host but the agent runs inside Docker, where the workspace parent
  is mounted at `/workspaces` and `ideas/` at `/app/ideas`. Previously it passed
  raw host paths (`str(self.work_dir)` / `str(self.idea_file)`) for `--workspace`
  and `--idea-file`; those don't exist in the container, so the agent would write
  its logs to an unmounted path (lost on container exit) — or, with a space in
  the host path like `chai lab`, fail argument parsing outright. Now it sends
  `/workspaces/<workspace_name>` and `/app/ideas/<rel_path>` instead.

### `templates/manager/system_prompt.txt`
- Expanded "do NOT engage the human" rules: don't ask about recoverable tool
  failures, infra/debug details, equivalent low-risk fixes, repeated
  confirmations, or anything that belongs in the session log.
- Added an explicit "engage the human ONLY when..." list: real research-scope
  decisions, destructive/irreversible/expensive/credentialed actions, info only
  the human can provide, or genuine preference-dependent forks.

### `docker/run.sh`
- Updated `interactive` usage and help text to mention the browser UI and the
  `--cli` flag.
- **Shell-quoted the agent's passthrough args in `cmd__run_agent`.** The handler
  builds the `docker run` invocation with `eval "... agent_runner.py $@"`. The
  unquoted `$@` is re-split on whitespace by `eval`, so a workspace path under
  `/Users/.../chai lab/...` broke at the space and `agent_runner.py` rejected the
  fragments (`unrecognized arguments`). Each arg is now run through
  `printf '%q'` before entering the `eval` string so spaces survive.
- **Mounted host `src/` into the agent container** (`-v "$PROJECT_ROOT/src:/app/src:ro"`
  in `cmd__run_agent`). Interactive mode runs each agent via
  `python /app/src/core/agent_runner.py` inside Docker, but `agent_runner.py` is
  newer than the published image and `src/` was never mounted — so the container
  used the stale baked-in `/app/src`, the file was missing, and every agent died
  with exit code 2 (`No such file or directory`). Because the agent never ran, it
  never wrote `workspaces/<ws>/logs/*.jsonl`, so the web transcript viewer had
  nothing to show. The read-only `src/` mount overlays current host source onto
  the image, so agents always run up-to-date code with no rebuild — and it won't
  recur as the interactive code keeps changing on this branch.
  > Note: this is a dev-time fix. For end users, the image must be rebuilt
  > (`./neurico build`) / republished so `agent_runner.py` is baked in. The other
  > (non-interactive) docker blocks also don't mount `src/`; they work only
  > because `runner.py` already exists in the published image.

## How it works (architecture)

```
                 InteractiveManager (agent loop)
                          │  send() / prompt() / poll_input()
                          ▼
                    UserChannel  ──────────────┐
                    /            \              │
           TerminalChannel    WebChannel        │
          (print/input)      (queues + SSE)     │
                                  │             │
                    InteractiveWebServer (thread)│
                       ├─ GET /        HTML page │
                       ├─ GET /stream  SSE  ◄────┘ conversation + status
                       ├─ POST /input  human reply
                       └─ agent-log tailer ──► workspaces/<ws>/logs/*.jsonl
                                                (formatted via visualizer.py)
```

- **Manager → human**: `WebChannel.send()` emits events to all SSE subscribers;
  history is replayed so late/reconnecting browsers see the full session.
- **Human → manager**: browser `POST /input` → `WebChannel.submit_input()` →
  inbound queue → unblocks `prompt()` / surfaces in `poll_input()`.
- **Agent transcript**: a tailer thread incrementally reads the workspace
  `.jsonl` logs and emits formatted `agentlog` events onto the same SSE stream.

## Usage

```bash
# Browser UI (default) — opens http://localhost:7890
./neurico interactive <idea_id> --provider claude

# Pick the web port / don't auto-open the browser
./neurico interactive <idea_id> --port 7895 --no-browser

# Original terminal interface
./neurico interactive <idea_id> --cli

# Standalone read-only log viewer for any workspace
python visualizer/visualizer.py <workspace_name>
```

## Cleanup before opening the PR

- [ ] Everything here is **uncommitted** working-tree changes — stage and commit.
- [ ] `visualizer/` contains a `.DS_Store` and `__pycache__/*.pyc` that should
      not be checked in (add to `.gitignore`).
