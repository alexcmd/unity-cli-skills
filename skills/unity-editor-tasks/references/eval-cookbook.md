# `eval` cookbook — running C# inside a live Editor

`unity command eval` compiles a C# snippet with Roslyn and runs it on the Editor's
**main thread**, returning the value. It reaches any API the project can — engine,
`UnityEditor`, the asset database, your own code. Use it when no registered command
fits. Verified against package `0.4.0-exp.1`.

Invoke (remember `--proxy-disable`):
```bash
unity command --proxy-disable --project-path <proj> eval --code "return Application.version;"
unity command --proxy-disable --project-path <proj> eval_file --file /abs/path/snippet.cs
```
Or via MCP `tools/call` / `POST /api/exec`: `{"command":"eval","parameters":{"code":"...","timeout":5000}}`.

## Rules that actually matter

- **Return a value with `return <expr>;`.** The returned object is JSON-serialized
  into the result. No `return` → `result` is null.
- **Multi-statement is fine** — write several statements; the `return` supplies the
  result: `var s = Application.dataPath; return s.Length;`.
- **Main thread**: you may call editor/engine APIs directly (no dispatch needed).
  Heavy work blocks the Editor for that tick — keep snippets quick or raise
  `timeout` (ms, default 5000).
- **Namespaces**: `UnityEngine` is effectively in scope; **fully-qualify
  `UnityEditor`** types (`UnityEditor.EditorApplication`, `UnityEditor.AssetDatabase`)
  or a `using` at the top of an `eval_file`. When unsure, fully-qualify.
- **Reading the result is double-wrapped** (see SKILL.md): value at `result.result`;
  compile/runtime errors in `result.diagnostics` with inner `result.success:false`.
  A transport `success:true` does NOT mean your C# compiled — check the inner one.
- **No implicit Undo.** Registered commands make Undo steps; `eval` mutations don't
  unless you call `UnityEditor.Undo.*`. Prefer registered commands for anything you
  might want to undo.

## Snippets

**Inspect the live scene**
```csharp
return UnityEngine.SceneManagement.SceneManager.GetActiveScene().GetRootGameObjects().Length;
```
```csharp
var go = GameObject.Find("Hero");
return go == null ? "missing" : go.transform.position.ToString();
```

**Create & configure (when you need logic a command can't express)**
```csharp
var go = GameObject.CreatePrimitive(PrimitiveType.Cube);
go.name = "Spawned";
go.transform.position = new Vector3(0, 5, 0);
var rb = go.AddComponent<Rigidbody>();
rb.mass = 3f;
return go.GetInstanceID();
```

**Batch edit across the scene**
```csharp
int n = 0;
foreach (var r in Object.FindObjectsByType<Renderer>(FindObjectsSortMode.None)) {
    r.enabled = true; n++;
}
return $"enabled {n} renderers";
```

**Undo-friendly mutation**
```csharp
var go = GameObject.Find("Hero");
UnityEditor.Undo.RecordObject(go.transform, "Move Hero");
go.transform.position = Vector3.zero;
return "moved (undoable)";
```

**Editor/asset operations**
```csharp
UnityEditor.AssetDatabase.SaveAssets();
return "assets saved";
```
```csharp
return string.Join(",", UnityEditor.EditorBuildSettings.scenes.Select(s => s.path));
```

**Play-mode control & queries**
```csharp
return UnityEditor.EditorApplication.isPlaying;
```
```csharp
UnityEditor.EditorApplication.isPaused = true; return "paused";
```

**Call your own project API** (the real power — no integration needed)
```csharp
return MyGame.Economy.GetBalance("player1");
```

## Longer scripts: `eval_file`

For anything past a few statements, write a `.cs` file and run it with `eval_file`
— you get `using` directives, helper methods, and readability. The file is compiled
and executed the same way; end with a `return`.

```csharp
// /tmp/spawn_grid.cs
using UnityEngine;
var parent = new GameObject("Grid").transform;
for (int x = 0; x < 5; x++)
for (int z = 0; z < 5; z++) {
    var c = GameObject.CreatePrimitive(PrimitiveType.Cube);
    c.transform.SetParent(parent);
    c.transform.position = new Vector3(x, 0, z) * 1.5f;
}
return 25;
```

## Safety

- `eval` runs unrestricted C# — it's gated by the descriptor's `evalToken` (the CLI
  handles this automatically; for raw HTTP the token goes in the `parameters.token`
  for `eval`/`exec`). Treat it like a live console: only against projects you trust.
- Snippets that spin (infinite loop, blocking IO) freeze the Editor until `timeout`.
- Prefer a registered command when one exists — typed args, validation, Undo.
