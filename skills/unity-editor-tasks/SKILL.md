---
name: unity-editor-tasks
description: >-
  Get real work done inside a live Unity Editor from the terminal or an AI agent —
  build scenes, spawn/edit GameObjects, add components, create prefabs and C#
  scripts, set serialized fields, enter Play mode, run tests, trigger builds,
  capture screenshots, and read the console — by calling the Pipeline package's
  ~140 registered commands, and running arbitrary C# with `eval` (a Roslyn REPL
  on the Editor's main thread) for anything not covered. Trigger this whenever the
  user wants to accomplish a concrete Unity task via the terminal/agent rather
  than in the GUI: "create a cube/player/enemy in the scene", "add a Rigidbody",
  "wire up this component", "spawn N objects", "make a prefab", "run the tests and
  fix failures", "screenshot the game view", "inspect/modify the live scene",
  "execute C# in the editor to do X", or automating gameplay/level/asset edits on
  a running Editor or dev Player. For installing the Pipeline package, connecting,
  MCP setup, and the proxy/HTTP details use `unity-editor-automation`; for headless
  builds/tests use `unity-build-test`; for editor/project management use `unity-cli`.
---

# Solving tasks in a live Unity Editor

Once the Pipeline package is running in an open Editor (setup + connection is the
**unity-editor-automation** skill), you drive the project two ways:

1. **Registered commands** — ~140 built-ins covering most editor operations.
   Structured, typed, single Undo step, safe. **Prefer these.**
2. **`eval` C#** — compile & run arbitrary C# on the Editor's main thread via
   Roslyn. The escape hatch for anything the commands don't cover.

A task is: *discover what's there → act (command first, eval if needed) → verify.*

Everything here goes through the `unity` CLI and a reachable Pipeline server. Run
first, fall back on failure: **if a command fails with command-not-found** (exit
127) the CLI isn't installed — use unity-cli's *install-on-demand* flow (ask the
user before installing); if `unity pipeline list` shows no reachable server, get
one running via **unity-editor-automation**.

## Non-negotiable: `--proxy-disable`

On a machine with an OS/PAC proxy, every live-Editor call **hangs** without it
(the CLI routes loopback through the proxy). Always pass `--proxy-disable` to
`unity command`, and keep it in the MCP server args. If a call hangs for its whole
timeout, this is why. (Full root-cause in unity-editor-automation.)

## Three ways to invoke a command

Same command, three transports — pick by context:

```bash
# 1. CLI — best for scalar args
unity command --proxy-disable --project-path <proj> create_gameobject --name Hero --primitive cube

# 2. eval via CLI
unity command --proxy-disable --project-path <proj> eval --code "return UnityEngine.Application.unityVersion;"
```
```jsonc
// 3. MCP tools/call OR POST /api/exec — best for nested/array/object args
{ "command": "set_transform",
  "parameters": { "target": "/Hero", "position": [0,1,0], "scale": [2,2,2] } }
```

For **complex args** (arrays, nested objects, `ObjectRef`), prefer MCP `tools/call`
or `POST /api/exec` with a JSON `parameters` object — cleaner than escaping JSON on
a CLI flag. See unity-editor-automation → `references/pipeline-http-api.md` for the
raw HTTP recipe and token/port discovery.

## Addressing objects: `ObjectRef` handles

Many commands take a `target`/`parent`/`source` of type **`ObjectRef`** — a handle
that can be any of:
- **hierarchy path** — `"/Hero"`, `"/Root/Enemy"` (leading `/`, scene-root relative)
- **instanceId** — the numeric id from `get_scene_hierarchy`/`find_gameobjects`
- **asset path** — `"Assets/Prefabs/Enemy.prefab"`
- **guid** / **globalId** — stable references

When unsure, call `get_scene_hierarchy` or `find_gameobjects` first and use the
returned `instanceId` or `hierarchyPath`.

## Reading results (envelope shapes)

Every call returns a success/result envelope. Two things to check:

- **Command failure**: top-level `success:false` (or MCP `isError:true`). A
  command can return HTTP 200 yet have failed logically — always check the flag.
- **`eval` is double-wrapped**: the transport reports `success:true` if the HTTP
  round-trip worked; the *real* C# outcome is the **inner** envelope. The returned
  value is at `result.result`; compile/runtime errors are in `result.diagnostics`
  and the inner `result.success:false`. Don't treat a 200 as "the C# worked."

```jsonc
// eval "return 6*7;" →
{ "success": true, "command": "eval",
  "result": { "success": true, "result": 42, "diagnostics": [], "executionTimeMs": 614 } }
```

## Method for solving a task

1. **Orient** — `get_scene_hierarchy`, `find_gameobjects`, `get_console_logs`,
   `editor_status` (is it compiling / in Play mode?). Don't act blind.
2. **Prefer a registered command.** Skim `references/command-catalog.md` (or live
   `unity command --proxy-disable --project-path <proj>` with no name) for one that
   fits. Structured + Undo-friendly beats hand-rolled eval.
3. **Fall back to `eval`** for logic the commands don't express — batch edits,
   custom queries, calling your own project APIs. See `references/eval-cookbook.md`.
4. **Verify** — re-read hierarchy, `get_console_logs severity=error`, or
   `capture_game_view` to a PNG. State what you changed.
5. **Persist** — `save_scene` after scene edits (they're in-memory until saved).

## Quick recipes

```bash
U="unity command --proxy-disable --project-path <proj>"

# Scene & objects
$U get_scene_hierarchy
$U create_gameobject --name Ground --primitive plane
$U create_gameobject --name Hero --primitive capsule
# set_transform/create_gameobjects with positions → use tools/call or /api/exec (array args)

# Components & fields
$U add_component --target /Hero --type Rigidbody
# set_component_properties / set_serialized_field: nested 'properties'/'value' → /api/exec

# Scripts & prefabs
$U create_script --name PlayerController --path Scripts
$U attach_script --target /Hero --type PlayerController      # after it compiles
$U create_prefab --source /Hero --path Prefabs/Hero.prefab

# Play, test, build, capture
$U editor_play    ;  $U editor_stop
$U run_tests --mode editor
$U capture_game_view --save_path Screenshots/frame.png
$U get_console_logs --severity error --limit 50

# Anything else — eval
$U eval --code "UnityEditor.EditorApplication.isPaused = true; return \"paused\";"
```

Full task-grouped catalog with real arg schemas: **`references/command-catalog.md`**.
C# `eval` idioms, gotchas, and worked snippets: **`references/eval-cookbook.md`**.

## Cautions

- **Mutations are real and immediate.** Scene/asset edits change the project. Most
  commands make one Undo step; `eval` does not unless you call `Undo.*` yourself.
  Say what you're about to change on anything destructive.
- **`save_scene` / `AssetDatabase.SaveAssets`** — in-memory edits are lost on
  Editor close until saved.
- **After creating a script**, it must compile before `attach_script --type` sees
  it. Importing a `.cs` (or `recompile`) triggers a **domain reload** — see below;
  poll `recompile_status` until `completed`, then `attach_script`. Or attach by
  `--script <path>`.
- **Check `editor_status.playMode` BEFORE structural edits.** Scene changes made in
  **Play mode are reverted on `editor_stop`** — so an edit + `save_scene` done while
  playing silently vanishes. Someone (or you) may have left the Editor in Play mode
  between calls, so never assume; if it's `playing` and you want a persistent edit,
  `editor_stop` first, then edit in edit mode, then `save_scene`. (Tell: if the
  Rotator/animations look mid-motion in a "static" capture, you're in Play mode.)
- **Domain reloads drop the connection and rotate the auth token.** Entering *or*
  exiting Play mode, and every `recompile`/script import, restart the Pipeline
  server — the in-flight call may return "connection reset"/"cannot connect", and
  the descriptor's `evalToken` changes. The CLI and MCP tools **auto-reconnect on
  the next call** (just retry `editor_status`). Raw-HTTP scripts must **re-read
  `Library/Pipeline/.unity-pipeline-port` for the new port+token after any reload**
  — a token cached before Play/recompile will silently 401 every request after.
- **Leaving Play mode**: if you `editor_play`, remember to `editor_stop`.
- **`build` / `switch_build_target`** require `confirm:true` (they're guarded);
  `dry_run:true` validates first. For pure headless builds prefer `unity build`
  (the unity-build-test skill).
