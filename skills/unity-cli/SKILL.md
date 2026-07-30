---
name: unity-cli
description: >-
  Use the `unity` command-line tool (Unity Hub CLI) to manage Unity from the
  terminal — install/uninstall/list editors and modules, create/open/register
  projects, browse project templates, inspect licenses and auth, configure
  Unity Cloud, and run environment diagnostics (doctor/env/status). Trigger this
  whenever the user mentions the `unity` CLI, "Unity Hub" from the command line,
  installing or switching Unity editor versions, creating a Unity project from
  the terminal, listing Unity releases/templates/modules, Unity licensing or
  sign-in, installing/upgrading the `unity` CLI binary itself, or wants
  terminal-based Unity project/editor management — even if they
  don't name the exact subcommand. For building/testing projects headlessly use
  the `unity-build-test` skill; for driving a live Editor (Pipeline, commands,
  MCP) use `unity-editor-automation`.
---

# Unity CLI (`unity`)

`unity` is Unity's official command-line front end to Unity Hub and installed
Editors. It manages editor installs, projects, templates, licenses, auth, Unity
Cloud, and environment diagnostics. This skill covers the management surface;
two sibling skills cover building/testing and live-Editor automation.

Version this reference was written against: **1.0.0-beta.3**. Confirm with
`unity --version`; flags evolve between beta builds, so when something looks off,
run `unity <cmd> --help`.

**Layer model** (from Unity's own framing): the CLI *manages* Unity (editors,
projects, auth), the `com.unity.pipeline` package *drives* a running Editor, and
`unity command eval` *reaches inside* it to run live C#. This skill is the first
layer; the driving/eval layers live in the **unity-editor-automation** skill.

## Install-on-demand: run first, install only if the CLI is missing (with consent)

**Don't gate work behind an upfront check — just run the `unity` command the task
needs.** Only when a call fails *specifically because the CLI isn't installed* do
you fall back to installing it, and you **ask the user to accept the install
first** — it pipes a script from the internet and drops a binary on their machine,
so never auto-install without consent.

### Recognize the "not installed" failure

A missing CLI surfaces as a **shell** command-not-found, not a `unity` error:

| Shell | Symptom | Exit code |
|-------|---------|-----------|
| bash / zsh | `unity: command not found` | **127** |
| PowerShell | `The term 'unity' is not recognized…` | — |

Any *other* failure (a real message to stderr, exit `1`) means the CLI **is**
installed — that's a normal `unity` error; handle it on its own terms and do
**not** reinstall.

### The fallback (only on a confirmed command-not-found)

1. **Stop and ask the user.** Tell them the `unity` CLI isn't installed, show the
   exact install command for their platform, and get an explicit yes before
   running it. If they decline, stop and tell them what to install to proceed.
2. **Install** (source: Unity's *"Meet the Unity CLI"* post —
   https://unity.com/blog/meet-the-unity-cli):
   ```bash
   # macOS / Linux
   curl -fsSL https://public-cdn.cloud.unity3d.com/hub/prod/cli/install.sh | UNITY_CLI_CHANNEL=beta bash
   ```
   ```powershell
   # Windows (PowerShell)
   $env:UNITY_CLI_CHANNEL='beta'; irm https://public-cdn.cloud.unity3d.com/hub/prod/cli/install.ps1 | iex
   ```
3. **Refresh the current shell, verify, then retry the original command** (the
   install dir depends on OS — see below):
   ```bash
   [ -f "$HOME/.unity/env" ] && . "$HOME/.unity/env"        # macOS: source the env file it added
   hash -r 2>/dev/null
   command -v unity >/dev/null || export PATH="$HOME/.local/bin:$HOME/.unity/bin:$PATH"
   unity --version          # confirm, then re-run what the user originally asked for
   ```

**Verified against the real `install.sh`** (a ~520-line bash script from Unity's
CDN that SHA-256-verifies its download; needs `curl`/`wget` + `shasum`/`sha256sum`):

- **Install location depends on OS.** **Linux** → `~/.local/bin/unity` ("xdg"
  layout; that dir is on `PATH` out of the box, no shell-config edit). **macOS** →
  `~/.unity/bin/unity` ("home" layout) plus a `~/.unity/env` sourced from your
  shell config. Override the directory with `UNITY_CLI_HOME=<dir>` (forces the
  home layout on any platform).
- **`UNITY_CLI_CHANNEL` accepts only `beta`, `alpha`, or empty (= stable)** — the
  literal string `stable` is **rejected** by the script (`omit` the var for
  stable). Use **`beta`** for now: the stable manifest (`latest.json`) currently
  404s, so a plain (channel-less) install can fail until a stable release ships.
- A **new** shell picks `unity` up automatically; only the *current* shell needs
  the refresh above. If an interactive login is ever required, have the user run
  the install themselves with a leading `! ` in the prompt.

Update later with `unity upgrade` (`--channel stable|beta`, `--rollback`). On a
fresh machine this one binary is all you need — it installs editors, modules, and
handles auth itself.

## Project problems surface as errors — react, don't pre-gate

Same philosophy for the project: don't pre-validate — run the command and let its
error tell you what to fix (most are self-healing):

- `Not a Unity project (ProjectVersion.txt not found …)` → wrong directory or not a
  Unity project. Run from the project root, or pass a registered name/path.
- **Editor version not installed** → add `--allow-install` to `build`/`test`/`run`/
  `open`, or run `unity projects require --allow-install` (installs the version
  from `ProjectVersion.txt`; safe no-op if already present).
- **License / auth errors** → `unity license activate …` / `unity auth login …`
  (see the licensing/auth sections below).

## Global conventions (apply to every command)

- **Output format**: `--format human|json|tsv|ndjson`, or `--json` shorthand.
  Human output is tab-separated tables. **When you need to parse output, always
  pass `--json`** — never scrape the human tables.
- **Streams & exit codes** (the CLI is built for automation): results go to
  **stdout**, errors to **stderr**, and exit codes follow a simple contract —
  **`0` success, `1` error, `130` cancelled**. Branch on the exit code in scripts
  rather than grepping messages; it's a stable contract, the human text isn't.
- **`--no-banner`** suppresses the startup banner. Set `UNITY_NO_BANNER=1` in the
  environment when running several commands so output stays clean.
- **`--non-interactive`** disables all prompts (for CI). It also changes behavior:
  auto-includes child modules, hard-errors instead of prompting for EULAs, and
  skips confirmations. `UNITY_NON_INTERACTIVE=1` is the env equivalent.
- **`--quiet`** suppresses informational chatter; **`--verbose`** adds full stack
  traces on failure.
- **Proxy**: `--proxy <url>` (http/https/socks/pac), `--proxy-disable`. The CLI
  also reads OS proxy settings automatically.
- Most env vars mirror flags: `UNITY_FORMAT`, `UNITY_ARCHITECTURE`,
  `UNITY_EDITOR_VERSION`, `UNITY_PROJECT_PATH`, `UNITY_CLOUD_ORG`.
- Many commands accept **short version aliases**: `6.5.3f1` or `6.2` resolve to a
  full version like `6000.5.3f1` where the context is unambiguous.

## First move: orient yourself

Before acting, check state. These are all read-only:

```bash
unity doctor          # platform, CLI version, install paths, auth, editors, proxy, recent logs
unity env             # Hub env paths (user data, editor install path, cache, config, proxy)
unity status          # live connected Editors (usually empty unless one is running)
unity editors -i      # installed editors (Version, Alias, Arch, Default, Platforms)
unity auth status     # who is signed in
unity license status  # active licenses and expiry
```

`unity doctor` is the single best "what's my setup" command and its output is
redaction-safe enough to reason from. For support hand-off use `unity diagnose
proxy` (redacted) rather than pasting raw logs.

## Command map

| Area | Commands |
|------|----------|
| Diagnostics | `doctor`, `env`, `status`, `diagnose`, `logs`, `changelog`, `--version` |
| Editors | `editors` (`-i` installed, `-r` releases, `default`, `running`, `info`, `path`, `upgrade`), `install`, `uninstall`, `releases` |
| Modules | `install-modules`, `modules list <ver>`, `editors module {list,add,remove,refresh}` |
| Install paths | `install-path -g/-s`, `editors install-path` |
| Projects | `projects` (`list`, `info`, `add`, `remove`, `create`, `new`, `clone`, `open`, `pin`/`unpin`, `require`, `size`, `link`/`unlink`, `upgrade`, `export`/`import`), top-level `open` |
| Templates | `templates` (`list`, `info`, `create`, `edit`, `delete`, `location`) |
| Licensing | `license` (`list`, `status`, `activate`, `return`, `server`) |
| Auth / Cloud | `auth` (`login`, `status`, `logout`), `cloud` (`status`, `org`, `project`) |
| Build / Test / Run | `build`, `test`, `run` → see **unity-build-test** skill |
| Live Editor | `pipeline`, `command`/`cmd`, `list`, `mcp` → see **unity-editor-automation** skill |
| Hub / CLI mgmt | `hub install`, `cache {info,clean}`, `config`, `analytics`, `language`, `completion`, `upgrade`, `self-uninstall`, `shell`, `bug` |

Full flag-by-flag details for every subcommand are in
`references/command-reference.md` — read it when you need exact options for a
command not spelled out below.

## Editors: install, list, switch, remove

```bash
unity releases --lts --limit 20          # browse installable versions (--stream alpha|beta|lts|tech, --since 2023)
unity editors -r                          # releases available to download
unity editors -i                          # what's installed
unity editors info 6000.5.3f1             # changeset + installer URL for a version
unity install 6000.5.3f1                  # install (interactive picks module set)
unity install 6000.5.3f1 -m android ios --cm --accept-eula   # non-interactive with modules + child modules
unity install <ver> --list-components     # list module ids without installing
unity install <ver> --dry-run             # show what would download
unity editors default 6000.5.3f1          # set the default editor (no arg = show current)
unity uninstall 6000.5.3f1 -y             # remove an editor
```

Key `install` flags: `-m/--module <ids...>`, `--cm/--childModules` (or
`--no-cm`), `-f/--force` reinstall, `-y/--yes`, `--accept-eula`, `--resume`
(resume interrupted downloads), `-a/--architecture x86_64|arm64`. Module ids come
from `unity install <ver> --list-components` or `unity modules list <ver>`.

Add an editor Unity Hub doesn't know about: `unity editors add <path>`.

## Projects

```bash
unity projects list                       # registered projects (optional [pattern])
unity projects info [pathOrName]          # rich project detail; defaults to CWD, or pass a name/path
unity projects add <paths...>             # register existing project folders in the Hub
unity projects require                    # ensure the project's editor version is installed (installs if missing)
unity projects size --all                 # disk usage across all registered projects
unity open [project]                      # open in the correct editor version (CWD default)
```

`unity projects info` needs a real Unity project (a `ProjectVersion.txt`); run it
from inside the project or pass a registered name. It reports editor version,
render pipeline, scripting backend, packages, cloud/VCS links, and more.

**Creating projects** — three entry points:
- `unity projects new <name>` — non-interactive, CI-friendly. Flags: `--path`,
  `--editor-version` (accepts `latest`/`lts`), `--template`, `--open`.
- `unity projects create <name>` — richer: everything `new` has plus
  `--cloud`/`--cloud-org`/`--cloud-project` to create+link a Unity Cloud project,
  and `--vcs github|gitlab` to create+link a git repo (with `--git-*` flags for
  namespace, visibility, token via `--git-token-stdin`, LFS, initial commit).
- `unity projects clone --vcs github|gitlab|uvcs --vcs-namespace <n> --vcs-repo <r>`
  — clone a repo and register the Unity project inside it.

Prefer `new` for scripted/CI creation; reach for `create` only when you also need
cloud or VCS linking. `--template` takes a package id like `com.unity.template.3d`.

## Templates

```bash
unity templates list -e 6000.5.3f1              # all templates for an editor version
unity templates list -e <ver> -t core           # filter: core|learning|sample|custom|new|all
unity templates list -i                          # only locally installed
unity templates info <name>
unity templates create <project-path>            # make a custom template from an existing project
```

Template `Status` in the listing is one of `ready`/`downloadable`/`upgradable`.
The `Package` column (e.g. `com.unity.template.urp-blank`) is what you feed to
`--template` when creating a project.

## Licensing, auth, cloud

```bash
unity license status                      # active licenses + expiry
unity license activate --personal --accept-eula   # free Personal license
unity license activate --serial <serial>          # serial-based
unity license activate --floating                 # from configured floating server
unity license server status                       # floating server seats
unity auth login                          # browser OAuth; or --client-id/--client-secret for a service account (CI)
unity auth status / logout
unity cloud status                        # cloud sign-in + active org
unity cloud org list / set-default <id|name>
unity cloud project list                  # projects in the active org (--cloud-org to override)
```

Auth and cloud share the same Unity account session. A service-account login
(`--client-id`/`--client-secret`) is the CI-friendly path.

## CLI housekeeping

```bash
unity cache info / clean          # download cache location/size; clean to reclaim space
unity config proxy [url]          # view/set proxy; unity config update-check on|off
unity analytics status            # 'false' = telemetry off (opt-in only); opt-in/opt-out to change
unity language --set en           # display language
unity upgrade --check             # check for a newer CLI; --channel stable|beta, --rollback
unity completion zsh              # print shell completion script
unity shell                       # warm REPL — many commands in one process (faster for batches)
unity bug --title ... --description ...   # file a bug to Unity's reporter
```

For repeated automation, `unity shell --protocol ndjson` exposes a
machine-readable request/response protocol over stdio, and the REPL avoids
per-command process startup cost.

## Working style

- Run `unity doctor` first when a user reports something "not working" — it
  surfaces auth, install paths, editor list, and proxy in one shot.
- Reach for `--json` the moment you need to branch on output; parse that, not the
  tables.
- Installs, uninstalls, and `cache clean` change the system — state the version
  and modules you're about to install and let the user confirm unless they've
  told you to proceed. `--dry-run` (install/uninstall/upgrade) previews safely.
- Never put keystore passwords or git tokens directly on the command line in a
  form that lands in shell history — use the `--*-stdin` variants or env vars.
