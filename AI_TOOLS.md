# AI tools in Codex

This file is a codebase-driven inventory of the tools the model can use in this repository,
plus the app-server RPC methods or request/notification surfaces that expose or support them.

## Authoritative sources

The most useful source files are:

- `codex-rs/core/src/tools/spec.rs` — authoritative built-in tool inventory, feature gating,
  handler registration, and aliases
- `codex-rs/core/src/tools/handlers/` — implementation of the built-in tool handlers
- `codex-rs/app-server/README.md` — JSON-RPC method names and request/notification flows
- `codex-rs/app-server-protocol/src/protocol/v2.rs` — typed protocol definitions

## How tool exposure works

There are three different ways the LLM gets tools in this codebase:

1. **Built-in tools**
   - Declared in `codex-rs/core/src/tools/spec.rs`
   - Registered into the runtime with `builder.register_handler(...)`
   - Used inside conversation turns started with `turn/start`

2. **Runtime-added MCP tools**
   - Added from configured MCP servers at runtime
   - Exposed to the model as normal function tools during a turn
   - Discovery/auth is handled through app-server MCP endpoints

3. **Runtime-added dynamic tools**
   - Supplied by the client in `thread/start.params.dynamicTools`
   - Executed through the `item/tool/call` request/response flow
   - Experimental API

## Important caveat

Not every tool is always available. `codex-rs/core/src/tools/spec.rs` gates tools by model,
feature flags, session source, and shell backend. So the list below is the full known surface in
this repository, not the guaranteed toolset for every run.

## Built-in tools the LLM can use

For most built-in tools, there is **no separate app-server RPC method per tool**. They are made
available inside a turn, so the relevant API surface is usually:

- `thread/start` / `thread/resume` / `thread/fork` — establish the conversation context
- `turn/start` / `turn/steer` — start or continue the turn where the model can call tools
- `item/*` notifications — stream the resulting tool activity back to the client

### Shell and filesystem tools

| Tool name        | What it does                                                                         | API endpoint / surface                                                                                                     |
| ---------------- | ------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------- |
| `shell`          | Executes an argv-style shell command.                                                | Available during `turn/start` / `turn/steer`; no dedicated tool RPC.                                                       |
| `shell_command`  | Executes a shell script string in the user's shell.                                  | Available during `turn/start` / `turn/steer`; no dedicated tool RPC.                                                       |
| `local_shell`    | Model-native local shell tool variant when `ConfigShellToolType::Local` is selected. | Included in the turn tool list; no separate app-server RPC.                                                                |
| `container.exec` | Compatibility alias registered to the same shell handler as `shell`.                 | Same as `shell`; alias only, not a distinct RPC method.                                                                    |
| `exec_command`   | Unified exec/PTTY-backed command execution that can return a session id.             | Available during `turn/start` / `turn/steer`; conceptually related to direct `command/exec`, but this tool is turn-scoped. |
| `write_stdin`    | Sends input to an existing `exec_command` session and polls for more output.         | Available during `turn/start` / `turn/steer`; conceptually related to direct `command/exec/write`.                         |
| `apply_patch`    | Applies a verified patch to files.                                                   | Available during `turn/start` / `turn/steer`; file approval flow uses `item/fileChange/requestApproval` when needed.       |
| `read_file`      | Reads a local file with line-number and indentation-aware modes.                     | Available during `turn/start` / `turn/steer`; no dedicated tool RPC.                                                       |
| `list_dir`       | Lists directory entries with offsets and depth controls.                             | Available during `turn/start` / `turn/steer`; no dedicated tool RPC.                                                       |
| `grep_files`     | Searches files by regex and returns matching paths.                                  | Available during `turn/start` / `turn/steer`; no dedicated tool RPC.                                                       |
| `view_image`     | Reads a local image from disk for the model.                                         | Available during `turn/start` / `turn/steer`; no dedicated tool RPC.                                                       |
| `update_plan`    | Records a structured plan/checklist for clients to render.                           | Available during `turn/start` / `turn/steer`; emitted to clients through turn events rather than a separate RPC method.    |

### JavaScript / code-execution tools

| Tool name       | What it does                                                                                     | API endpoint / surface                                               |
| --------------- | ------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------- |
| `js_repl`       | Runs raw JavaScript in a persistent Node kernel with top-level await.                            | Available during `turn/start` / `turn/steer`; no dedicated tool RPC. |
| `js_repl_reset` | Resets the persistent JS kernel state.                                                           | Available during `turn/start` / `turn/steer`; no dedicated tool RPC. |
| `exec`          | Code mode: runs raw JavaScript in a `node:vm` context and can call nested tools from `tools.js`. | Available during `turn/start` / `turn/steer`; no dedicated tool RPC. |
| `artifacts`     | Runs JavaScript against the preinstalled artifact runtime to build presentations/spreadsheets.   | Available during `turn/start` / `turn/steer`; no dedicated tool RPC. |

### User-interaction and approval tools

| Tool name             | What it does                                                                                          | API endpoint / surface                                                                                                                                        |
| --------------------- | ----------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `request_user_input`  | Asks the user one to three short questions and waits for answers.                                     | Tool call is turn-scoped; the client-side request flow uses `item/tool/requestUserInput`, and app-server also documents experimental `tool/requestUserInput`. |
| `request_permissions` | Requests extra permissions from the user, then reuses granted permissions for later shell-like calls. | Tool call is turn-scoped; approval transport uses `item/permissions/requestApproval`.                                                                         |

### Multi-agent orchestration tools

| Tool name                 | What it does                                            | API endpoint / surface                                               |
| ------------------------- | ------------------------------------------------------- | -------------------------------------------------------------------- |
| `spawn_agent`             | Starts a sub-agent for a bounded task.                  | Available during `turn/start` / `turn/steer`; no dedicated tool RPC. |
| `send_input`              | Sends a follow-up message to an existing spawned agent. | Available during `turn/start` / `turn/steer`; no dedicated tool RPC. |
| `resume_agent`            | Reopens a previously closed agent.                      | Available during `turn/start` / `turn/steer`; no dedicated tool RPC. |
| `wait`                    | Waits for one or more agents to finish.                 | Available during `turn/start` / `turn/steer`; no dedicated tool RPC. |
| `close_agent`             | Closes an agent and returns its last status.            | Available during `turn/start` / `turn/steer`; no dedicated tool RPC. |
| `spawn_agents_on_csv`     | Runs one worker agent per CSV row.                      | Available during `turn/start` / `turn/steer`; no dedicated tool RPC. |
| `report_agent_job_result` | Worker-only callback tool used by CSV agent jobs.       | Available only to worker contexts; no dedicated app-server RPC.      |

### MCP resource helper tools

These are built-in helpers for **MCP resources**. They are separate from runtime MCP tools.

| Tool name                     | What it does                                   | API endpoint / surface                                                                                                                      |
| ----------------------------- | ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| `list_mcp_resources`          | Lists resources from configured MCP servers.   | Turn-scoped tool; supporting management/discovery RPCs include `mcpServerStatus/list` and `config/mcpServer/reload`.                        |
| `list_mcp_resource_templates` | Lists parameterized MCP resource templates.    | Turn-scoped tool; supporting management/discovery RPCs include `mcpServerStatus/list` and `config/mcpServer/reload`.                        |
| `read_mcp_resource`           | Reads a specific MCP resource by server + URI. | Turn-scoped tool; supporting management/discovery RPCs include `mcpServerStatus/list` and `mcpServer/oauth/login` for auth-enabled servers. |

### Model-native tools emitted from `ToolSpec`

These are not backed by Rust tool handlers in `core/src/tools/handlers/`, but they are still part
of the model tool surface built in `codex-rs/core/src/tools/spec.rs`.

| Tool name          | What it does                                                                           | API endpoint / surface                                       |
| ------------------ | -------------------------------------------------------------------------------------- | ------------------------------------------------------------ |
| `web_search`       | Native web search tool spec, with cached/live external web access depending on config. | Included in the turn tool list; no dedicated app-server RPC. |
| `image_generation` | Native image generation tool spec (PNG output).                                        | Included in the turn tool list; no dedicated app-server RPC. |

### Internal / test-only tool

| Tool name        | What it does                                      | API endpoint / surface                                                                       |
| ---------------- | ------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| `test_sync_tool` | Synchronization helper used by integration tests. | Present only when experimental/test tooling is enabled; not intended for normal product use. |

## Runtime-added tool surfaces

### MCP tools (runtime-added)

Configured MCP servers can contribute additional **tool-callable functions** at runtime. These are
converted into regular function tools and registered in the same tool builder loop that registers
built-in tools.

Relevant API surfaces:

- `mcpServerStatus/list` — lists configured MCP servers, tools, resources, templates, and auth state
- `mcpServer/oauth/login` — starts OAuth login for a configured MCP server
- `mcpServer/oauthLogin/completed` — notification when that login flow finishes
- `config/mcpServer/reload` — reloads MCP server configuration
- `mcpServer/elicitation/request` — request/response flow when an MCP server asks the client for structured input

### Dynamic tools (runtime-added, experimental)

Dynamic tools are supplied by the client and become callable by the LLM for a single thread.

Relevant API surfaces:

- `thread/start.params.dynamicTools` — declares the tool list for the thread
- `thread/resume` / `thread/fork` — can continue working with that thread context
- `item/tool/call` — server-to-client JSON-RPC request when the model invokes a dynamic tool
- `DynamicToolCallParams` / `DynamicToolCallResponse` in `app-server-protocol/src/protocol/v2.rs`

Example flow from the repository docs:

1. Client starts a thread with `dynamicTools`.
2. Model invokes one of those tools during a turn.
3. Server sends `item/tool/call` to the client.
4. Client returns `contentItems` and `success`.

## Direct command RPCs that look tool-like, but are not LLM turn tools

These methods are app-server utilities that a client can call directly without involving a model
turn. They matter because they overlap with the shell tooling surface, but they are **not** the
same thing as the LLM's built-in tool calls.

| RPC method                 | Purpose                                                             |
| -------------------------- | ------------------------------------------------------------------- |
| `command/exec`             | Run one standalone command under the app-server sandbox.            |
| `command/exec/write`       | Write stdin bytes or close stdin for a running direct exec session. |
| `command/exec/resize`      | Resize a PTY-backed direct exec session.                            |
| `command/exec/terminate`   | Terminate a running direct exec session.                            |
| `command/exec/outputDelta` | Stream stdout/stderr chunks for a direct exec session.              |

## Practical summary

If you want the shortest map of “what tools can the LLM use?” in this repo:

- **Core built-ins:** declared in `codex-rs/core/src/tools/spec.rs`
- **Handlers for built-ins:** `codex-rs/core/src/tools/handlers/`
- **Per-turn app-server entrypoint:** `turn/start`
- **Dynamic tool transport:** `item/tool/call`
- **User-input transport:** `item/tool/requestUserInput`
- **Permission transport:** `item/permissions/requestApproval`
- **MCP discovery/auth:** `mcpServerStatus/list`, `mcpServer/oauth/login`, `config/mcpServer/reload`
- **Direct non-LLM exec RPCs:** `command/exec*`

## Maintenance notes

When this file needs to be refreshed, re-check these places in order:

1. `codex-rs/core/src/tools/spec.rs`
2. `codex-rs/core/src/tools/handlers/mod.rs`
3. `codex-rs/core/src/tools/spec.rs` handler registration calls
4. `codex-rs/app-server/README.md`
5. `codex-rs/app-server-protocol/src/protocol/v2.rs`
