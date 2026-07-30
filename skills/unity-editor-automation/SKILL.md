---
name: unity-editor-automation
description: >-
  Drive a live Unity Editor from the terminal or from an AI agent via the `unity`
  CLI's Pipeline package — list and execute custom Editor commands
  (`unity command`, `unity list`), run live C# inside a running Editor or Player
  with `unity command eval` (a Roslyn REPL), install/upgrade the
  `com.unity.pipeline` package, and configure Unity as an MCP server for AI
  clients (`unity mcp`, `unity mcp configure`). Trigger this whenever the user
  wants to run a Unity Editor command/tool from the command line, execute/evaluate
  C# in a running Unity instance, inspect or mutate a live scene from the terminal,
  connect Claude/Cursor/VS Code/Codex to Unity over MCP, set up the Unity MCP
  server, invoke a `[CliCommand]`/registered Editor action headlessly, drive a
  built dev Player at runtime (`--runtime`), talk to a running Editor's
  HTTP/Pipeline server, or automate an open Editor/Player instance — even if they
  don't name the exact subcommand. For headless builds/tests use `unity-build-test`; for installing
  editors, creating projects, or licensing use `unity-cli`.
---

# Automating a live Unity Editor

The Unity **Pipeline** package (`com.unity.pipeline`, **experimental**; works with
**Unity 6.0 LTS and newer**) runs a small localhost HTTP server inside a running
Editor — or inside a **development Player build**. The CLI talks to it to list and
execute commands the project registers, to run arbitrary C# with `eval`, and it's
the same surface that backs Unity's MCP server for AI agents. Written against CLI
**1.0.0-beta.3**.

**Preflight:** this all runs through the `unity` CLI — if `command -v unity` fails,
install it first via the check-then-install preflight in the **unity-cli** skill.

This is the "drive it / reach inside it" layer of the CLI (the "manage it" layer —
installs, projects, auth — is the **unity-cli** skill). The three capabilities, in
increasing power: **registered commands** (operations you anticipated) →
**`eval`** (arbitrary C# you didn't) → **MCP** (all of it exposed to an AI agent
as callable tools). The agent loop it enables is *observe a live project → act on
it → verify the result*, with no human relaying console output.

## The connection model

```
unity CLI  ──HTTP──▶  Pipeline server  (inside a running Unity Editor
   │                                     with the project open)
   └── unity mcp ──stdio──▶ MCP client (Claude, Cursor, VS Code, Codex, …)
```

Every `command`/`list`/MCP call needs **all** of:
1. A Unity Editor **running with the project open**.
2. The **`com.unity.pipeline` package installed** in that project.
3. The Pipeline **HTTP server reachable**.

If any is missing you get:
`Error: No Unity Editor instances found with reachable Pipeline servers.`
Diagnose with `unity pipeline list` (shows each instance's Running / Pipeline /
Server Reachable / Safe Mode state).

**If a live-Editor command hangs, it's almost certainly the proxy — add
`--proxy-disable`.** On a machine with an OS/PAC proxy configured (check
`unity doctor` → `proxy.*`), `unity command`, `unity list`, `unity pipeline list`,
`unity status`, and the `unity mcp` bridge **all hang for their full timeout**
even though the Pipeline server is healthy. Root cause (diagnosed on CLI
`1.0.0-beta.3`): the CLI routes its **loopback** `127.0.0.1` request through the
proxy instead of honoring the `localhost,127.0.0.1,::1` noProxy bypass, so the
request stalls. Raw `curl`/Node to the same port return in milliseconds because
they go direct. The fix is to bypass the proxy for these loopback calls:

```bash
unity command --proxy-disable --project-path <proj> editor_status   # 0.3s vs hang
```

For MCP, add `--proxy-disable` to the server's args (in Claude Code:
`~/.claude.json` → `mcpServers.unity-editor-mcp.args`, right after `mcp`) — then
`tools/list` returns the full catalog instantly. `--proxy-disable` is scoped to
that invocation and is exactly right for loopback traffic; leave the global proxy
alone so external calls (install/releases) keep working.

**Still stuck? Drive the HTTP API directly.** The server is a plain loopback API:
read the port + bearer token from `<project>/Library/Pipeline/.unity-pipeline-port`
and hit `http://127.0.0.1:<port>/api/*`. Routes, auth rules, the ~140-command
catalog, and copy-paste recipes are in **`references/pipeline-http-api.md`** — also
where you get the exact per-command arg schema (`GET /api/commands`).

## Step 1 — install the Pipeline package

```bash
unity pipeline list-versions                 # available package versions (latest flagged)
unity pipeline install                        # install latest into the project (auto-detects path)
unity pipeline install --project-path /path/to/Proj
unity pipeline install --package-version 0.4.0-exp.1
unity pipeline install --force                # re-resolve/update even if present
unity pipeline upgrade                        # upgrade to the newest available
```

`unity pipeline list` reports, per Editor instance: Project, Path, PID, Running,
Pipeline (installed?), Version, Update Available, Server Port, Server Reachable,
Safe Mode. Use it to confirm the server is up before issuing commands.

## Step 2 — list and run Editor commands

```bash
unity list                                    # tools the Pipeline package exposes on the connected Editor
unity command                                 # list available commands (omit the command name)
unity command <name> [args...]                # execute a command
unity command <name> --timeout 60 -- <args>   # args after -- go to the command
```

Locator flags (shared by `command`, `list`, `mcp`): `--project-path <path>` (env
`UNITY_PROJECT_PATH`; auto-detected from CWD otherwise), `--runtime <player exec
name>` and `--runtime-path <path>` to target a built **Player** runtime instead of
the Editor. `unity command` also takes `--timeout <seconds>` (default 30).

Commands are **not a fixed set** — any static method in the project becomes one by
adding a `[CliCommand]` attribute, with `[CliArg]` on its parameters. The package
**auto-discovers** them (no registration step), so `unity command` lists whatever
*that* project exposes. This self-describing quality is exactly why an AI agent can
discover what it can do at runtime instead of relying on a hardcoded list.

```csharp
using Unity.Pipeline.Commands;
using UnityEngine;

public static class MyPipelineCommands
{
    [CliCommand("greet", "Log a greeting and return its length")]
    public static int Greet(
        [CliArg("name", "Who to greet", Required = true)] string name)
    {
        Debug.Log($"Hello, {name}!");
        return name.Length;
    }
}
```

Then `unity command greet --name World` runs it against the connected Editor and
hands back the result (`4`). Each such command maps naturally onto the kind of tool
a function-calling model or MCP server already knows how to invoke.

### Headless one-shot: `unity run --command`

To run a registered command *without* an Editor already open, use the build/run
path (also in the **unity-build-test** skill):

```bash
unity run --command MyTool.Generate -- --count 50 --seed 7
```

This spawns the Editor in batch mode, waits for the Pipeline server, runs the
command with args after `--` parsed against its `[CliArg]` schema, prints the
return value, and shuts down — or reuses an already-open Editor. `--format json`
wraps the return value in a result envelope; `Debug.Log` output streams to stderr.

## Live C# with `eval` — the REPL into a running project

Registered commands cover the operations you anticipated; **`unity command eval`
covers the ones you didn't.** It compiles an arbitrary C# expression with Roslyn
and runs it on the Editor's **main thread**, so it can reach any engine or editor
API the project can — rendering, physics, animation, the asset database, the
Editor itself, plus your own code. Answers come back in **milliseconds** against an
already-running instance, versus the seconds of an edit→recompile→relaunch cycle.
No predefined command, no plugin, no recompile, no domain reload.

```bash
unity command eval "return Application.version;"
unity command eval "return UnityEditor.EditorApplication.isPlaying;"
unity command eval "var s = Application.dataPath; return s.Length;" --json
unity command eval_file "path/to/script.cs"        # run a whole C# file
unity command eval "..." --runtime MyGame          # same power, aimed at a live Player
```

- Write `return <expr>;` to get a value back; multi-statement bodies work too
  (last `return` supplies the result). `eval_file` runs a longer script from disk.
- **Security token**: because `eval` runs unrestricted C#, it's gated behind a
  token — expect to configure/pass it before eval works (it's off by default for
  the same reason the runtime API is). Treat it like a live console into the
  process: never point it at anything you don't fully trust.
- This is what gives an AI agent a *tight* feedback loop: inspect the live scene,
  mutate it, re-enter Play mode, read the result — all without recompiling.

## Step 3 — expose Unity to AI agents over MCP

Bare `unity mcp` starts the **MCP stdio server** for the current project (pin a
different one with `--project-path`). AI clients spawn this to reach the Editor.

Write client config automatically:

```bash
unity mcp configure --list                    # supported clients + their config file paths
unity mcp configure claude                     # Claude Desktop
unity mcp configure claude-code                # Claude Code CLI
unity mcp configure cursor --local             # project-local config (clients that support it)
unity mcp configure vscode --dry-run           # preview the JSON without writing
unity mcp configure <client> --yes             # skip the "already exists, update?" prompt
```

Supported clients include: `claude` (Claude Desktop), `claude-code`, `cursor`,
`vscode` / `vscode-insiders` (GitHub Copilot), `copilot-cli`, `windsurf`, `cline`,
`codex` (OpenAI Codex CLI), `kiro`, `trae`, `openclaw`, `antigravity`, `zed`,
`continue`, `inspect` (MCP Inspector). Run `unity mcp configure --list` for the
current, machine-specific list and each client's config path.

`--local` writes project-scoped config (e.g. `.cursor/`, `.vscode/mcp.json`) for
clients that support it; without it, config goes to the user-global location.
Always safe to preview with `--dry-run` first.

## Driving a built dev Player at runtime

The Pipeline package isn't Editor-only. Drop its **runtime component into a
development build** and the running game exposes the same API — so a script, test
rig, or AI agent can drive a live gameplay session: pull live logs, query runtime
status, invoke registered commands, hot-reload code, or `eval` C# against the
running Player.

```bash
unity command --runtime MyGame                       # list commands the Player exposes
unity command --runtime MyGame set_camera_speed --speed 10
unity command eval "return Time.timeScale;" --runtime MyGame
```

- **`--runtime <name>`** locates the Player by **process name**;
  **`--runtime-path <path>`** points at the Player's runtime **port-descriptor
  file**. If a runtime call hangs or reports "no reachable Pipeline servers,"
  check you used the right one — a full `.exe` path belongs on `--runtime-path`,
  not `--runtime` (older CLI builds may tolerate an exe path on `--runtime`).
- **localhost-only and off by default** — intended for **dev and QA builds, never
  production**. It's a deliberate opt-in because it lets anything local drive the
  game.

## Typical end-to-end flows

**Wire Claude Code to a Unity project:**
```bash
unity pipeline install                        # 1. add the Pipeline package
# 2. open the project in the Editor (unity open — see unity-cli)
unity pipeline list                           # 3. confirm Server Reachable = true
unity mcp configure claude-code --dry-run     # 4. preview, then run without --dry-run
```

**Run a project tool from the terminal:**
```bash
unity pipeline list        # verify a reachable server
unity command              # discover command names
unity command BuildAtlas --timeout 120
```

## Troubleshooting

- **"No Unity Editor instances found with reachable Pipeline servers"** — the most
  common error. Check `unity pipeline list`: is an Editor Running, is Pipeline
  installed, is Server Reachable true? Open the project, `unity pipeline install`,
  and make sure it isn't in Safe Mode.
- **Safe Mode** (shown in `unity pipeline list`) — the project has compile errors;
  the Pipeline server won't come up until scripts compile. Fix compilation first.
- **Wrong instance** when several Editors are open — disambiguate with
  `--project-path`.
- **Targeting a built Player** rather than the Editor — use `--runtime <name>` or
  `--runtime-path <path>`.
- **Version drift** — `unity pipeline list` flags "Update Available"; run
  `unity pipeline upgrade`.
