# Pipeline command catalog — task-grouped, with real arg schemas

Captured live from `GET /api/commands` on package `0.4.0-exp.1` (140 commands).
Arg types: `String`, `Int32`, `Boolean`, `Single[]`=float array, `Single[][]`=array
of float arrays, `ObjectRef`=object handle (path/instanceId/guid/globalId/assetPath),
`ObjectId[]`=instanceId array, `JObject`/`JToken`=arbitrary JSON. **Always fetch
the live schema** for exact args of any command not listed here:
`unity command --proxy-disable --project-path <proj> --format json` (list), or
`GET /api/commands`.

Invocation: `unity command --proxy-disable --project-path <proj> <name> --<arg> <val>`
for scalars; for array/object args use MCP `tools/call` or `POST /api/exec` with a
JSON `parameters` object.

## Orient / observe
- `get_scene_hierarchy [path]` — GameObject tree of an open scene (nodes carry
  name, instanceId, hierarchyPath, components, children).
- `find_gameobjects [name][tag][type][hierarchy_path][include_inactive]` — filtered search.
- `list_open_scenes`, `get_selection`, `editor_status` (compiling/domainReload/playMode).
- `get_console_logs [severity=all|log|warning|error][limit≤1000]`, `console`, `clear_console`.
- `get_performance_stats`.

## GameObjects & transforms
- `create_gameobject [name][primitive: cube|sphere|capsule|cylinder|plane|quad][parent: ObjectRef]`
- `create_gameobjects [name][primitive][parent][count][positions: Single[][]][rotations: Single[][]][scales: Single[][]]`
  — batch; array lengths must equal `count`.
- `set_transform target:ObjectRef [position: Single[]][rotation: Single[] euler deg][scale: Single[]]`
  — omitted channels unchanged.
- `delete_gameobject`, `rename_gameobject`, `set_active`, `set_parent`, `set_layer`, `set_tag`, `set_selection`.

## Components & serialized data
- `add_component target:ObjectRef type:String` — e.g. `Rigidbody`, `UnityEngine.Camera`.
- `remove_component`, `get_component_properties target[,type]`.
- `set_component_properties target properties:JObject [type]` — `properties` maps
  serialized-property name → value (vectors/colors as arrays; object refs as ObjectRef). One Undo step.
- `set_serialized_field target field:String value:JToken [component]` — `field` is a
  SerializedProperty path: `speed`, `settings.speed`, `waypoints.Array.data[0]`. Object
  refs: pass an ObjectRef object as `value`.
- `get_serialized_fields target [field][component]`.

## Scripts
- `create_script name:String [path][namespace][base_class=MonoBehaviour][overwrite]`
  — path is authoring-root relative (`Assets/` optional).
- `attach_script target:ObjectRef [type][script]` — add a MonoBehaviour by compiled
  `type` name OR by `script` asset path. Use `--script <path>` to attach before/without
  a known compiled type.
- `recompile` (+ `recompile_status`) — force a script recompile and wait.

## Prefabs
- `create_prefab source:ObjectRef path:String` — save a GameObject as a prefab; source
  becomes a connected instance.
- `create_prefab_variant`, `instantiate_prefab prefab:ObjectRef [scene_path][name]`,
  `apply_prefab_overrides`, `revert_prefab_overrides`, `unpack_prefab`, `save_prefab_contents`.

## Scenes
- `open_scene path:String [additive]`, `create_scene`, `save_scene [path]`, `save_all`,
  `set_active_scene path:String` (new objects go into the active scene).

## Assets & files
- `create_asset`, `create_folder`, `import_asset`, `copy_asset`, `move_asset`,
  `rename_asset`, `delete_asset`, `find_assets`.
- `read_text_file`, `write_text_file`, `reload_file`.

## Play mode & tests
- `editor_play`, `editor_pause`, `editor_stop`, `editor_focus`, `set_autotick`.
- `run_tests [mode=all|editor|playmode][filter][filter_type=testName|assembly|category]`
  `[include_explicit][async_tests][timeout=300s]` — async_tests returns immediately;
  poll `test_status`.
- `list_tests [mode]`, `cancel_tests`.

## Build
- `build [target][outputPath][profileName][options: String[]][scenes: String[]][confirm][dry_run]`
  — **guarded**: needs `confirm:true` to run; `dry_run:true` validates. Async → poll `build_status`.
- `switch_build_target target:String [confirm]` — destructive (full reimport + domain reload).
- `list_build_targets`, `list_build_profiles`, `get_build_settings`, `set_build_settings`,
  `add_scene_to_build`, `remove_scene_from_build`.

## Capture / visual verification
- `capture_game_view [width=1280][height=720][camera][save_path][include_inline_image][max_resolution]`
  — renders a camera to PNG; inline base64 unless `save_path` set.
- `capture_scene_view`, `screenshot`.

## Baking
- `bake_lighting`, `bake_navmesh`, `bake_navmesh_surfaces`, `bake_occlusion_culling`
  (+ each has `*_status`, `cancel_*`, and `clear_*`).

## Animation & Timeline
- `create_animation_clip`, `create_animator_controller`, `add_animator_state`,
  `add_animator_layer`, `add_animator_parameter`, `add_animator_transition`,
  `set_animation_curve`, `remove_animation_curve`, `get_animation_clip`, `get_animator_controller`.
- `create_timeline`, `add_timeline_track`, `add_timeline_clip`, `get_timeline`.

## Project settings (get_*/set_* pairs)
`player`, `quality`, `graphics`, `physics`, `audio`, `time`, `input`, `lighting`,
`navmesh`, `tags_layers` — e.g. `get_player_settings` / `set_player_settings`.
Also `get_import_settings`/`set_import_settings`, `list_shaders`, `get_shader_properties`,
`get_material_properties`/`set_material_properties`.

## Packages
`package_add`, `package_remove`, `package_list`, `package_search`, `package_resolve`,
`package_status`.

## Escape hatch
- `eval code:String [timeout ms=5000]` — run C# via Roslyn (see `eval-cookbook.md`).
- `eval_file file:String [timeout]` — run a `.cs` file from disk.
- `menu` — invoke an Editor menu item by path.
