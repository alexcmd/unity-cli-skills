---
name: unity-build-test
description: >-
  Build, test, and batch-run Unity projects headlessly from the terminal with the
  `unity` CLI (`unity build`, `unity test`, `unity run`) — for CI pipelines, local
  automated builds, and command-line test runs. Trigger this whenever the user
  wants to compile/build a Unity project from the command line, produce a player
  for a platform (StandaloneWindows64/OSX, Android APK/AAB, iOS, WebGL), run
  EditMode/PlayMode tests headlessly, set up Unity CI (GitHub Actions, GitLab,
  Jenkins), sign an Android build from the CLI, or run a Unity project in batch
  mode — even if they don't name the exact subcommand. For installing editors,
  creating projects, or licensing use the `unity-cli` skill; for driving a live
  Editor's Pipeline commands use `unity-editor-automation`.
---

# Unity build, test & run from the CLI

Three commands spawn the Editor in batch mode: `unity build`, `unity test`,
`unity run`. They forward conventional CI flags and stream the Editor log. All
three default the `[project]` argument to the current directory and read the
editor version from `ProjectVersion.txt` (override with `--editor-version` or env
`UNITY_EDITOR_VERSION`).

Written against CLI **1.0.0-beta.3**; verify flags with `unity <cmd> --help`.

**If a `unity` command fails with command-not-found** (`unity: command not found`,
exit 127), the CLI isn't installed — fall back to unity-cli's *install-on-demand*
flow (ask the user to accept the install, then retry). Don't pre-check; just run
what you need and only install on that specific failure.

## Before anything: the license + editor preconditions

Batch builds need an **activated license** and the **project's editor version
installed**. In CI this is the usual failure point.

```bash
unity license status                 # confirm a usable license (Personal/Pro/Industry)
unity projects require               # ensure the project's editor version is installed (installs if missing)
```

- Add `--allow-install` to `build`/`test`/`run` to auto-install the editor version
  if absent. Otherwise install it first (`unity install <ver>` — see unity-cli).
- Activate a license non-interactively before building: `unity license activate
  --serial <serial>` (Pro), `--personal --accept-eula` (free), or `--floating`
  (floating server). For a service account, sign in with `unity auth login
  --client-id ... --client-secret ...`.

## Building — `unity build`

Unity has **no built-in command-line build entry point**, so you must point at a
static C# method that performs the build. `--target` and `--execute-method` are
both **required**.

```bash
unity build \
  --target StandaloneOSX \
  --execute-method BuildScript.PerformBuild \
  --output-path Builds/mac \
  --editor-version 6000.5.3f1 \
  --allow-install
```

Your `--execute-method` is a normal editor build method:

```csharp
// Assets/Editor/BuildScript.cs
public static class BuildScript
{
    public static void PerformBuild()
    {
        // -buildOutput arrives via the CLI's --output-path; read it yourself:
        var args = System.Environment.GetCommandLineArgs();
        // ... resolve output path from args, then:
        BuildPipeline.BuildPlayer(
            EditorBuildSettings.scenes,
            /* output */ "Builds/mac",
            BuildTarget.StandaloneOSX,
            BuildOptions.None);
    }
}
```

**Important:** `-o/--output-path` is passed to Unity as `-buildOutput`, but *your
executeMethod is responsible for honoring it*. The CLI can't control where a
custom build method writes. Same for versioning — the CLI passes hints; the method
applies them.

Core flags:
- `--target <t>` — StandaloneWindows64, StandaloneOSX, StandaloneLinux64, Android,
  iOS, WebGL, etc. **Required.**
- `--execute-method <Class.Method>` — **Required.**
- `--build-target-group <g>` — forwarded as `-buildTargetGroup`.
- `-o/--output-path <path>` — forwarded as `-buildOutput`.
- `-l/--log-file <path>` — default `<project>/Logs/build-<target>-<timestamp>.log`.
- `--editor-version` / `-e/--editor-path` / `-a/--architecture x86_64|arm64`.
- `--args "<string>"` — extra args passed to Unity (shell-split).
- `--no-tail` — don't stream the log to stdout (default streams live).
- `--allow-install` — install the editor version if missing.
- `--allow-dirty-build` — skip the uncommitted-changes guard (by default a dirty
  working tree blocks the build).

### Versioning

`--versioning-strategy semantic|tag|custom|none` (default `none`). With `custom`,
supply `--build-version <string>`; other strategies derive the version
themselves (`tag` from git tags, `semantic` from semver rules). The build method
must actually stamp `PlayerSettings.bundleVersion` etc. from the passed value.

### Android signing

Android-only flags (ignored for other targets):
- `--android-export-type apk|aab|android-studio-project` (default apk)
- `--android-keystore-base64 <content>` — base64 keystore, decoded to a temp file
  for the build and deleted after.
- `--android-keystore-password <pass>`, `--android-key-alias <alias>`,
  `--android-key-alias-password <pass>` (defaults to keystore password).
- `--android-target-sdk-version <N>`, `--android-symbol-type none|public|debugging`,
  `--android-version-code <N>`.

**Secrets warning:** CLI args land in shell history and CI logs. Prefer injecting
keystore material through CI secret env vars and passing them in, and never echo
them. The base64/temp-file handling exists so you don't keep a keystore on disk.

## Testing — `unity test`

```bash
unity test --mode EditMode --output test-results.xml
unity test --mode PlayMode --filter "MyNamespace.Combat" --allow-install
```

- `--mode EditMode|PlayMode` — omit to run the editor's default test platform.
- `--filter <pattern>` — run only tests whose names match.
- `--output <path>` — NUnit XML report (default `test-results.xml`). This is the
  file CI should collect and publish.
- `--editor-version` / `-e/--editor-path` / `-a/--architecture`.
- `--allow-install`, `--timeout <seconds>` (kills Unity after N seconds; off by
  default — always set one in CI so a hung editor fails the job instead of hanging).

The command exits non-zero when tests fail, so it drops straight into a CI gate.
Parse the NUnit XML for per-test detail.

## Batch running — `unity run`

Runs the project in batch mode and forwards args to the editor — for headless
tasks that aren't a full build or the test runner.

```bash
unity run --command MyTool.Generate -- --count 50 --seed 7
```

- `--command <name>` runs a registered `[CliCommand]` Editor command headlessly:
  args after `--` are parsed against that command's `[CliArg]` schema (no manual
  `Environment.GetCommandLineArgs()` parsing). Requires the `com.unity.pipeline`
  package — see the **unity-editor-automation** skill. A running Editor with the
  project open is reused; otherwise one is spawned and shut down.
- `--timeout <seconds>` (env `UNITY_RUN_TIMEOUT`), `--allow-install`,
  `--editor-version`/`-e`/`-a`.
- With `--format json` you get a result envelope containing the command's return
  value; the Editor log (including `Debug.Log`) streams to stderr.

## CI recipe (shape)

```bash
export UNITY_NO_BANNER=1
unity auth login --client-id "$UNITY_CLIENT_ID" --client-secret "$UNITY_CLIENT_SECRET"
unity license activate --serial "$UNITY_SERIAL"      # or --personal --accept-eula
unity projects require --allow-install || true       # ensure editor present
unity test --mode EditMode --output editmode.xml --timeout 1800
unity test --mode PlayMode --output playmode.xml --timeout 1800
unity build --target StandaloneLinux64 \
  --execute-method BuildScript.PerformBuild \
  --output-path Builds/linux --allow-install
unity license return                                 # free the seat when done
```

Return floating/serial licenses at the end of a CI job (`unity license return`) so
you don't leak seats.

## Gotchas

- **Missing execute method** is the #1 build failure — there is no default build;
  you must write and reference one.
- **Dirty working tree** blocks `build` unless `--allow-dirty-build`. In CI this
  is usually fine to allow since the checkout is pristine anyway.
- **No editor installed** → pass `--allow-install` or pre-install; otherwise the
  command errors immediately.
- **Hung editors**: always set `--timeout` on `test`/`run` in automation.
- **Log location**: builds write to `<project>/Logs/` by default — collect that
  plus the results XML as CI artifacts.
