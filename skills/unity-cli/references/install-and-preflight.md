# Install the CLI on demand + react to project errors

Detail behind the one-paragraph **Gate** in SKILL.md. Read this only when a
`unity` call fails with command-not-found, or a project command errors.

## Recognize "CLI not installed"

A missing CLI is a **shell** command-not-found, not a `unity` error:

| Shell | Symptom | Exit code |
|-------|---------|-----------|
| bash / zsh | `unity: command not found` | **127** |
| PowerShell | `The term 'unity' is not recognized…` | — |

Any *other* failure (a real message on stderr, exit `1`) means the CLI **is**
installed — a normal `unity` error; handle it, do **not** reinstall.

## Install (only after the user accepts)

Never auto-install — it pipes a script from the internet and drops a binary on the
machine. Ask first; if declined, stop and say what's needed. Source: Unity's
*"Meet the Unity CLI"* post — https://unity.com/blog/meet-the-unity-cli.

```bash
# macOS or Linux
curl -fsSL https://public-cdn.cloud.unity3d.com/hub/prod/cli/install.sh | UNITY_CLI_CHANNEL=beta bash
```
```powershell
# Windows
$env:UNITY_CLI_CHANNEL='beta'; irm https://public-cdn.cloud.unity3d.com/hub/prod/cli/install.ps1 | iex
```
(Note: Support for standard package managers like brew, winget and apt is coming
soon. Refer to the CLI documentation for the latest.)

Then refresh the current shell, verify, and **retry the original command**:
```bash
[ -f "$HOME/.unity/env" ] && . "$HOME/.unity/env"        # macOS: source the env file it added
hash -r 2>/dev/null
command -v unity >/dev/null || export PATH="$HOME/.local/bin:$HOME/.unity/bin:$PATH"
unity --version
```

**Verified against the real `install.sh`** (~520-line bash from Unity's CDN;
SHA-256-verifies its download; needs `curl`/`wget` + `shasum`/`sha256sum`):

- **Install location depends on OS.** **Linux** → `~/.local/bin/unity` ("xdg"
  layout; on `PATH` out of the box, no shell-config edit). **macOS** →
  `~/.unity/bin/unity` ("home" layout) + a `~/.unity/env` sourced from your shell
  config. `UNITY_CLI_HOME=<dir>` overrides the dir (forces home layout anywhere).
- **`UNITY_CLI_CHANNEL` accepts only `beta`, `alpha`, or empty (= stable)** — the
  literal `stable` is **rejected**; omit the var for stable. Use **`beta`** for
  now: the stable manifest (`latest.json`) currently 404s.
- A **new** shell picks `unity` up automatically; only the current shell needs the
  refresh above. If an interactive login is ever required, have the user run the
  install themselves with a leading `! ` in the prompt.

Update later with `unity upgrade` (`--channel stable|beta`, `--rollback`).

## Project errors — react, don't pre-gate

Run the command; its error tells you what to fix (most self-heal):

- `Not a Unity project (ProjectVersion.txt not found …)` → wrong directory or not a
  Unity project. Run from the project root, or pass a registered name/path.
- **Editor version not installed** → add `--allow-install` to `build`/`test`/`run`/
  `open`, or `unity projects require --allow-install` (installs the version from
  `ProjectVersion.txt`; safe no-op if present).
- **License / auth errors** → `unity license activate …` / `unity auth login …`.
