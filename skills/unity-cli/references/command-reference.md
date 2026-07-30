# Unity CLI — full command reference

Exhaustive per-command options captured from `unity <cmd> --help` (v1.0.0-beta.3).
Every command also accepts the global flags (`--format`, `--json`, `--no-banner`,
`--non-interactive`, `--quiet`, `--verbose`, `--proxy`, `--proxy-disable`,
`--log-proxy`) — those are omitted below to reduce noise.

## Contents
- [Diagnostics](#diagnostics)
- [Editors & installs](#editors--installs)
- [Modules](#modules)
- [Projects](#projects)
- [Templates](#templates)
- [Licensing](#licensing)
- [Auth & Cloud](#auth--cloud)
- [Build / Test / Run](#build--test--run)
- [Live Editor: pipeline, command, list, mcp](#live-editor)
- [CLI housekeeping](#cli-housekeeping)

---

## Diagnostics

**`unity doctor`** — `--tail <lines>` (recent log lines, default 20). Prints
platform, arch, node/cli version, install path, log dir, auth state, installed
editors, proxy, recent logs.

**`unity env`** — no options. Prints user data path, editor install path, download
cache path, config path, Hub version, proxy.

**`unity status`** — `--port <n>` filter one Editor; `--project-path <substr>`
filter by project path substring. Columns: Port, State, Project, Version, PID.

**`unity diagnose proxy`** — redacted proxy diagnostic report, also written to logs.

**`unity logs`** — `--tail <lines>` (default 20; 0 = all), `-f/--follow`,
`--level trace|debug|info|warn|error|fatal` (default info).

**`unity changelog`** — release notes for the installed CLI.

---

## Editors & installs

**`unity editors|e`** (top-level flags) — `-r/--releases`, `-i/--installed`,
`-a/--architecture x86_64|arm64`, `--verbose` (full module names/paths),
`-w/--watch`. Subcommands:
- `add <path...>` — register an existing editor with the Hub.
- `default [version]` — show or set the default editor.
- `list` — installed editors or available releases.
- `running` — running editors + the project each has open.
- `info <version>` — release details (changeset, support status, installer URL).
- `module` — see [Modules](#modules).
- `install-path|ip` — show/change global editor install path.
- `path <version>` — print install directory of an installed version.
- `upgrade [editor]` — upgrade an installed editor to the newest patch in its line.

**`unity install|i [version]`** — install an editor.
- `-a/--architecture x86_64|arm64` (env `UNITY_ARCHITECTURE`)
- `-c/--changeset <changeset>` (for archive installs)
- `-m/--module <ids...>`
- `--cm/--childModules` / `--no-cm/--no-childModules`
- `-f/--force` reinstall; `-y/--yes` auto-pick first match
- `--accept-eula`; `--dry-run`; `--resume` (resume interrupted downloads from cache)
- `--no-elevate` (Windows: skip UAC helper)
- `--list-components` — list module ids + downloader-cli names, then exit

**`unity uninstall|u [version]`** — `-a/--architecture`, `-y/--yes`. Version accepts
full (`6000.2.12f1`) or short alias (`6.2.12f1`, `6.2`); uses stored default if omitted.

**`unity releases`** — `--lts`, `--stream alpha|beta|lts|tech`, `--since <year>`,
`--limit <n>` (default 20), `--skip <n>` (pagination).

**`unity install-path|ip`** — `-s/--set <path>`, `-g/--get`.

---

## Modules

**`unity install-modules|im`** — install/list modules for an installed editor.
- `-e/--editor-version <version>`, `-m/--module <ids...>`, `-l/--list`, `--all`
- `-a/--architecture`, `--cm/--childModules` / `--no-cm`
- `--accept-eula`, `--dry-run`, `-y/--yes`
- `--reinstall` (repair), `-f/--force` (implies reinstall + child modules + skip prompts)
- `--no-elevate`, `--retries <n>` (retry failed download/validation with backoff)

**`unity modules list <version>`** — available modules for a version.

**`unity editors module`** subcommands: `list <version>`, `add <version>`,
`remove <version>`, `refresh <version>` (re-fetch the module list).

---

## Projects

**`unity projects|p`** subcommands:
- `list [pattern]` — registered projects.
- `add <paths...>` — register existing project folders.
- `remove <paths...>` — unregister (does NOT delete files).
- `info [pathOrName]` — detail for a local project (defaults to CWD).
- `create <name>` — full-featured creation (see below).
- `new <name>` — non-interactive/CI creation (see below).
- `clone` — clone a repo and register the project inside.
- `link` / `unlink` — connect/disconnect a project to its cloud or VCS link.
- `open [pattern]` — open by name, fuzzy title, or path.
- `pin <pattern>` / `unpin <pattern>` — favorite / unfavorite.
- `require [pathOrName]` — assert required editor version is installed (installs if needed).
- `size [project]` — disk usage by folder; `--all` summarizes every registered project.
- `upgrade [pathOrName]` — upgrade a project to a different editor version.
- `export` / `import [file]` — export/import the Hub project list as JSON.

**`unity projects new <name>`** — `--path`, `--editor-version` (accepts
`latest`/`lts`), `--template <name|package-id>`, `-a/--architecture`, `--open`.

**`unity projects create <name>`** — superset of `new`, plus:
- Cloud: `--cloud`, `--cloud-org <id|name>` (env `UNITY_CLOUD_ORG`),
  `--cloud-project <id|name>` (link existing instead of creating).
- VCS: `--vcs github|gitlab`, `--git-namespace`, `--git-repo`,
  `--git-visibility private|public|internal`, `--git-default-branch`,
  `--git-token <pat>` / `--git-token-stdin` (CI-safe), `--no-initial-commit`,
  `--git-lfs`, `--vcs-region`.

**`unity projects clone`** — `--vcs github|gitlab|uvcs`, `--vcs-namespace`,
`--vcs-repo`, `--ref <branch|sha|changeset>`, `--path <dest>`,
`--project-path <subpath>` (when repo holds multiple projects),
`--git-token`/`--git-token-stdin`, `--no-lfs`.

**`unity open [project]`** (top-level) — `--editor-version`, `-e/--editor-path`,
`-a/--architecture`, `--build-target <t>`, `--build-target-group <g>`,
`--args <string>` (extra args passed to Unity). Project can be a name, glob, or path.

---

## Templates

**`unity templates|t`** subcommands:
- `list` — `-e/--editor <version>`, `-i/--installed`,
  `-t/--type core|learning|sample|custom|new|all`, `--custom`.
- `info <name>`.
- `create <project-path>` — custom template from an existing project.
- `edit <name>` — edit a custom template's metadata.
- `delete <name>` — delete a user-generated template.
- `location` — get/set/reset the default custom-template storage path.

Listing columns: Package, Name, Type (CORE/LEARNING/SAMPLE), Version, Status
(ready/downloadable/upgradable).

---

## Licensing

**`unity license`** subcommands:
- `list` — active licenses (Product, Type, Organization, Expires).
- `status` — current license state + signed-in flag.
- `activate` — `--serial <serial>`, `--personal`, `--floating`, `--file <.ulf|.xml>`
  (offline), `--generate-request <.alf path>` (save offline activation request),
  `--accept-eula` (required with `--personal`).
- `return` — return the active licenses.
- `server` — floating license server: `status` (seats), `list` (configured servers).

---

## Auth & Cloud

**`unity auth|a`**: `login` (browser OAuth, or `--client-id`/`--client-secret` for
a service account), `status`, `logout`.

**`unity cloud`**:
- `status` — cloud sign-in + active org.
- `org` — `list`, `current`, `set-default <id|name>`, `clear-default`.
- `project list` — projects in the active org; `--cloud-org <id|name>` (env
  `UNITY_CLOUD_ORG`) overrides the active org.

---

## Build / Test / Run

Summarized here; the **unity-build-test** skill covers usage in depth.

**`unity build [project]`** — headless build. Required: `--target <t>` and
`--execute-method <Class.Method>` (Unity has no built-in CLI build). Also:
`--build-target-group`, `-o/--output-path`, `-l/--log-file`, `--editor-version`,
`-e/--editor-path`, `-a/--architecture`, `--args`, `--no-tail`, `--allow-install`,
Android signing (`--android-*`, keystore via base64), `--versioning-strategy
semantic|tag|custom|none`, `--build-version`, `--allow-dirty-build`.

**`unity test [project]`** — `--mode EditMode|PlayMode`, `--filter <pattern>`,
`--output <xml>` (default `test-results.xml`), `--editor-version`,
`-e/--editor-path`, `-a/--architecture`, `--allow-install`, `--timeout <s>`.

**`unity run [project]`** — batch-mode run. `--editor-version`, `-e/--editor-path`,
`-a/--architecture`, `--allow-install`, `--command <name>` (run a registered
`[CliCommand]` Editor command headlessly; args after `--` parse against its
schema), `--timeout <s>`.

---

## Live Editor

Summarized here; the **unity-editor-automation** skill covers usage in depth.
These require a running Editor with the `com.unity.pipeline` package and its HTTP
server reachable.

**`unity pipeline|pipe`**: `install [--project-path --force --package-version]`,
`upgrade`, `list` (instances + Pipeline status), `list-versions`.

**`unity command|cmd [command] [args...]`** — execute a command on a connected
Editor (omit command to list). `--project-path`, `--runtime <player exec name>`,
`--runtime-path <path>`, `--timeout <s>` (default 30).

**`unity list`** — list tools registered by the Pipeline package on the connected
Editor. `--project-path`, `--runtime`, `--runtime-path`.

**`unity mcp`** — start MCP stdio server (bare `unity mcp`), or
`configure [client]` to write MCP config for an AI client. Flags on `configure`:
`--list`, `--local`, `--yes`, `--dry-run`. Locator flags: `--project-path`,
`--runtime`, `--runtime-path`.

---

## CLI housekeeping

**`unity cache`**: `info`, `clean` (empties the download cache).

**`unity config`**: `proxy [url]` (view/set), `update-check on|off`.

**`unity analytics`**: `status`, `opt-in`, `opt-out`. Off by default; opt-in only.
`UNITY_NO_CONSENT_PROMPT=1` suppresses the one-time consent prompt without
recording a choice (for wrapper scripts).

**`unity language|lang`**: `--set <code>` (en, fr, de, es, ja_jp, ko_kr, pt_br, ru,
zh_cn, zh_tw).

**`unity hub install`** — install the Unity Hub application.

**`unity upgrade`** — `--check`, `--changelog`, `-y/--yes`, `--channel stable|beta`,
`--target <version>`, `--rollback`, `--dry-run`.

**`unity self-uninstall`** — `-y/--yes`, `--purge` (also remove user data),
`--dry-run`.

**`unity shell`** — interactive REPL; `--protocol ndjson` for machine-readable
request/response over stdio.

**`unity completion <shell>`** — print a completion script.

**`unity bug`** — `--email`, `--title`, `--description`, `--steps <steps...>`,
`--reproducibility first-time|sometimes|always`.
