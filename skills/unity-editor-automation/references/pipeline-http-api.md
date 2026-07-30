# Pipeline HTTP API (direct) — and the CLI-hang fallback

The `unity command` / `unity list` / `unity pipeline list` CLI wrappers talk to a
small **loopback HTTP server** the `com.unity.pipeline` package runs inside a
running Editor or dev Player. When those wrappers misbehave (see below), you can
drive the exact same server directly with `curl`. This was verified against
package **0.4.0-exp.1** + CLI **1.0.0-beta.3** on macOS.

## When to reach for this

**First, rule out the proxy.** On a machine with an OS/PAC proxy (`unity doctor` →
`proxy.*`), the live-Editor CLI commands (`unity command`, `list`, `pipeline
list`, `status`, `mcp`) **hang for their full timeout** because the CLI routes its
`127.0.0.1` request through the proxy instead of honoring the loopback noProxy
bypass (diagnosed on CLI 1.0.0-beta.3). The primary fix is **`--proxy-disable`**
on those commands (and in the MCP server args) — that makes them return in a
fraction of a second. Reach for the direct HTTP API below when `--proxy-disable`
isn't enough, or when you want the raw response / exact per-command schema. The
HTTP API always responds in **milliseconds** because `curl`/Node dial the loopback
directly and never touch the proxy.

## Discovery: the port descriptor file

The Editor server writes a descriptor the moment it starts. Read it to get the
port and the auth token — no CLI needed:

- **Editor:** `<project>/Library/Pipeline/.unity-pipeline-port` (fixed path)
- **Runtime/Player:** `.unity-pipeline-runtime-port` next to the Player (Windows:
  beside the `.exe`; macOS: inside the `.app` bundle)

```json
{
  "pid": 12771, "port": 7800, "mode": "editor",
  "projectPath": ".../QuadoGame", "projectName": "QuadoGame",
  "unityVersion": "6000.5.3f1",
  "evalToken": "FOWVlSpkoV/LYt9Gc5v4Bsjup1fpM0LU6gD7UrzqKZQ="
}
```

`evalToken` is the **bearer token** for *every* request (not just `eval`).

**Re-read it after any domain reload.** Entering/exiting Play mode and every
recompile/script-import restart the server and **rotate the token** (and possibly
the port). A token you read before `editor_play`/`recompile` will **silently 401
every subsequent request** — the classic symptom is a capture/exec loop that
"stops working" mid-run. Re-read the descriptor after each reload (or on the first
401). This bit a frame-capture-during-Play loop: 0 frames until the token was
re-read post-Play.

**Port ranges:** Editor `7800–7849` (tests `7850–7899`), Runtime `7900+`. The
server also logs `Pipeline Server started on port 7801` to the interactive Editor
log at `~/Library/Logs/Unity/Editor.log` (macOS) — note that's **not** the
project's `Logs/Editor.log`, which is the batch/import log.

## Connection rules

- **Dial `127.0.0.1`, never `localhost`.** The server binds IPv4 loopback; Unity's
  Mono `HttpListener` answers `::1` (IPv6) requests with `400`, and `localhost`
  resolves to `::1` non-deterministically → intermittent failures.
- Send `Authorization: Bearer <evalToken>`. Missing/wrong token → `401`.
- **Do not send an `Origin` header** — the server rejects any request carrying one
  (it blocks browsers/CORS; legitimate CLI clients never send it).

## Routes

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/api/status` | server heartbeat (`ready`) |
| GET | `/api/editor_status` | compiling / domainReload / playMode / version |
| GET | `/api/commands` | full command catalog (names, args, descriptions) |
| POST | `/api/exec` | run a command — body `{"command","parameters":{…}}` |
| GET | `/api/test-status` | test-run status |

Unknown path → `404`. `exec` body over the size cap → `413`.

## Recipe

```bash
P=/path/to/Project
TOK=$(python3 -c "import json;print(json.load(open('$P/Library/Pipeline/.unity-pipeline-port'))['evalToken'])")
PORT=$(python3 -c "import json;print(json.load(open('$P/Library/Pipeline/.unity-pipeline-port'))['port'])")
H=(-H "Authorization: Bearer $TOK" -H "Content-Type: application/json")
BASE="http://127.0.0.1:$PORT"

curl -s "${H[@]}" "$BASE/api/editor_status" | python3 -m json.tool
curl -s "${H[@]}" "$BASE/api/commands"      | python3 -m json.tool

# Run a read command
curl -s "${H[@]}" -X POST "$BASE/api/exec" \
  -d '{"command":"get_scene_hierarchy","parameters":{}}' | python3 -m json.tool

# Live C# via eval (params: code [required], timeout ms [default 5000])
curl -s "${H[@]}" -X POST "$BASE/api/exec" \
  -d '{"command":"eval","parameters":{"code":"return UnityEngine.Application.unityVersion;"}}'
```

Response envelope: `{ "success": true, "command": "...", "result": {...} }`. For
`eval`, the inner `result.result` holds the returned value and
`result.executionTimeMs` the timing; compile errors surface in
`result.diagnostics`.

## Built-in command catalog (~140 in 0.4.0-exp.1)

Far more than a starter set — the Editor is deeply scriptable out of the box.
Groups (representative, not exhaustive):

- **Lifecycle/observe:** `editor_status`, `editor_play`/`pause`/`stop`,
  `editor_focus`, `get_performance_stats`, `get_console_logs`, `console`,
  `clear_console`, `screenshot`, `capture_game_view`, `capture_scene_view`.
- **Scenes:** `get_scene_hierarchy`, `open_scene`, `create_scene`, `save_scene`,
  `save_all`, `list_open_scenes`, `set_active_scene`.
- **GameObjects/components:** `create_gameobject(s)`, `delete_gameobject`,
  `set_transform`, `set_parent`, `add_component`, `remove_component`,
  `get/set_component_properties`, `get/set_serialized_field(s)`, `set_active`,
  `set_layer`, `set_tag`, `rename_gameobject`, `find_gameobjects`, `set_selection`.
- **Prefabs:** `create_prefab`, `create_prefab_variant`, `instantiate_prefab`,
  `apply/revert_prefab_overrides`, `unpack_prefab`, `save_prefab_contents`.
- **Assets/files:** `create_asset`, `create_folder`, `create_script`,
  `attach_script`, `import_asset`, `copy/move/rename/delete_asset`, `find_assets`,
  `read_text_file`, `write_text_file`, `reload_file`.
- **Animation/Timeline:** `create_animation_clip`, `create_animator_controller`,
  `add_animator_state`/`layer`/`parameter`/`transition`, `set/remove_animation_curve`,
  `create_timeline`, `add_timeline_track`/`clip`.
- **Baking:** `bake_lighting`, `bake_navmesh(_surfaces)`, `bake_occlusion_culling`
  (+ `*_status` / `cancel_*` / `clear_*`).
- **Build/tests:** `build`, `build_status`, `list_build_targets`/`profiles`,
  `switch_build_target`, `get/set_build_settings`, `add/remove_scene_from_build`,
  `run_tests`, `list_tests`, `cancel_tests`, `test_status`.
- **Project settings:** `get/set_player_settings`, `quality`, `graphics`,
  `physics`, `audio`, `time`, `input`, `lighting`, `navmesh`, `tags_layers`.
- **Packages:** `package_add`/`remove`/`list`/`search`/`resolve`/`status`.
- **Escape hatch:** `eval`, `eval_file`, `recompile`, `menu` (invoke a menu item).

Fetch the live, exact schema (arg names/types per command) from
`GET /api/commands` for the connected project — it's authoritative and
version-specific.
