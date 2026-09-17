# Starter Skeleton

Use this file only when drafting or heavily rewriting service lifecycle scripts. Adapt service names, actions, commands, paths, ports, and process-identity checks to the repository; do not copy placeholders literally.

- [Dispatcher](#dispatcher)
- [Per-Service Script](#per-service-script)
- [Flutter Web CLI Package](#flutter-web-cli-package)
- [Optional Extensions](#optional-extensions)

For a multi-service repository, prefer this layout:

```text
scripts/
  setup.sh
  services/
    api.sh
    web.sh
  lib/                  # Only when shared mechanics justify it
```

## Dispatcher

Keep `scripts/setup.sh` focused on validation, missing-value prompts, and dispatch:

```bash
#!/usr/bin/env bash
set -euo pipefail

ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"
SERVICE_DIR="${ROOT}/scripts/services"
readonly ROOT SERVICE_DIR

readonly -a ACTIONS=(start stop restart run status)
readonly -a SERVICES=(api web)

usage() {
  printf 'Usage: %s <action> <service>\n' "${0##*/}" >&2
}

die() {
  printf 'error: %s\n' "$*" >&2
  exit 2
}

contains() {
  local needle="$1"
  shift
  local item
  for item in "$@"; do
    [[ "${item}" == "${needle}" ]] && return 0
  done
  return 1
}

choose() {
  command -v gum >/dev/null 2>&1 ||
    die "missing argument and gum is unavailable; run with explicit arguments"
  gum choose "$@"
}

service_script_for() {
  case "$1" in
    api) printf '%s\n' "${SERVICE_DIR}/api.sh" ;;
    web) printf '%s\n' "${SERVICE_DIR}/web.sh" ;;
    *) return 1 ;;
  esac
}

dispatch() {
  local action="$1"
  local service="$2"
  local script

  script="$(service_script_for "${service}")"
  [[ -f "${script}" ]] || die "missing service script: ${script}"
  bash "${script}" "${action}"
}

main() {
  (( $# <= 2 )) || {
    usage
    die "too many arguments"
  }

  local action="${1:-}"
  local service="${2:-}"

  [[ -n "${action}" ]] || action="$(choose "${ACTIONS[@]}")"
  contains "${action}" "${ACTIONS[@]}" || {
    usage
    die "unknown action: ${action}"
  }

  [[ -n "${service}" ]] || service="$(choose "${SERVICES[@]}")"
  contains "${service}" "${SERVICES[@]}" || {
    usage
    die "unknown service: ${service}"
  }

  dispatch "${action}" "${service}"
}

main "$@"
```

This baseline assumes every listed service supports every listed action, and that none of them take a `dev`/`prod` argument — a host only ever runs one active install per service, so `start`/`stop`/`restart`/`run`/`status` act on whatever is currently installed. If capabilities differ per service, add an explicit action-to-service matrix, filter interactive service choices, and reject unsupported combinations before dispatch. Add `install-dev`/`install-prod`/`publish` per [Optional Extensions](#optional-extensions) only when `setup.sh` owns those steps; none takes a separate environment argument, since the action name already fixes which one it is (`install-prod` optionally takes a version instead).

## Per-Service Script

Let each service script own its paths, ports, runtime files, concrete commands, and PID lifecycle:

```bash
#!/usr/bin/env bash
set -euo pipefail

SERVICE_NAME="api"
CLI_NAME="funflix"       # the service's own installed CLI, see service-release-governance
PORT=8080
CONFIG_PATH=""            # optional override; empty means rely on the CLI's own default path
                           # (${XDG_CONFIG_HOME:-~/.config}/<org>/funflix/config.toml). Either
                           # way, "${CLI_NAME}" itself writes and reads funflix.pid next to
                           # whichever config path is actually in effect — this script never
                           # touches that file directly.

readonly SERVICE_NAME CLI_NAME PORT CONFIG_PATH

usage() {
  printf 'Usage: %s <start|stop|restart|run|status>\n' "${0##*/}" >&2
}

die() {
  printf 'error: %s\n' "$*" >&2
  exit 2
}

cli_args() {
  # Both a locally-built install and a registry-pinned install expose the
  # same CLI; do not branch this on dev/prod.
  CLI_ARGS=(--port "${PORT}")
  # Must be `if`, not `[[ ... ]] && CLI_ARGS+=(...)`: under `set -e`, that
  # form's exit status is the `[[ ]]` test itself when CONFIG_PATH is empty,
  # which is 1 (false) — and being the function's last command, that 1
  # propagates as cli_args's own return status and aborts the whole script,
  # silently breaking start/run/restart whenever CONFIG_PATH is unset.
  if [[ -n "${CONFIG_PATH}" ]]; then
    CLI_ARGS+=(--config "${CONFIG_PATH}")
  fi
}

do_start() {
  cli_args
  "${CLI_NAME}" server start "${CLI_ARGS[@]}"
}

do_run() {
  cli_args
  exec "${CLI_NAME}" server run "${CLI_ARGS[@]}"
}

do_stop() {
  "${CLI_NAME}" server stop
}

do_restart() {
  do_stop
  do_start
}

do_status() {
  "${CLI_NAME}" server status
}

main() {
  local action="${1:-}"

  case "${action}" in
    start|stop|restart|run|status)
      (( $# == 1 )) || {
        usage
        die "${action} takes no further arguments"
      }
      "do_${action}"
      ;;
    *)
      usage
      die "unknown action: ${action:-<empty>}"
      ;;
  esac
}

main "$@"
```

`do_start`/`do_run`/`do_stop`/`do_status` are thin delegations: the installed CLI backgrounds itself (`server start`), stays foregrounded when asked (`server run`), and owns the PID file that `server stop`/`server status` read — this script does not `nohup` anything, write a PID file, or poll for liveness itself. If a given repository's CLI genuinely cannot daemonize itself, fall back to the `nohup`-and-PID-file pattern in [runtime-patterns.md](runtime-patterns.md#backgrounding-pattern) instead of this primary shape; in that fallback, terminate via `funshell port <port> --kill` rather than hand-signaling the PID file's PID when `funshell` is available (falling back to PID-based `TERM` plus a repository-specific identity check — stable executable, command marker, or runtime metadata — only when it isn't), and never hand-roll a separate port-based discovery-then-kill outside of `funshell`.

For a true single-service repository, keep the same validation and lifecycle boundaries in `scripts/setup.sh` and omit the dispatcher layer.

## Flutter Web CLI Package

Use this skeleton only for a Flutter web service that has no daemonizing CLI of its own yet, per [runtime-patterns.md](runtime-patterns.md#providing-the-missing-cli-a-self-published-npm-wrapper). This package is published via ordinary `npm publish`, independently of the `funbuild`/`funpub` pipeline that builds and distributes the web bundle it serves.

```text
apps/<app>/
├── <Flutter source>
└── extbuild/
    └── npm-cli/
        ├── package.json
        ├── bin/cli.js
        └── src/
            ├── daemonize.js
            ├── static-server.js
            └── proxy.js
```

```json
{
  "name": "<app>-cli",
  "version": "0.1.0",
  "bin": { "<app>-cli": "bin/cli.js" },
  "files": ["bin", "src"]
}
```

Do not set `"private": true` — it blocks `npm publish` unconditionally. Keep `dependencies` empty; use Node's standard library for daemonizing, static serving, and proxying.

```js
#!/usr/bin/env node
const { start, run, stop, status } = require("../src/daemonize");

const [, , group, action, ...rest] = process.argv;

const handlers = { start, run, stop, status };

if (group !== "server" || !handlers[action]) {
  console.error(`Usage: ${require("../package.json").name} server <start|run|stop|status>`);
  process.exit(2);
}

handlers[action].call(undefined, rest);
```

```js
// src/daemonize.js (sketch — adapt PID path, args, and grace period to the repository)
const GRACE_PERIOD_MS = 500;

function start(args) {
  // 1. reject a live PID file; report/remove a stale one
  // 2. spawn the actual server as a detached child (stdio ignored / redirected to a log file)
  // 3. write the child PID to an XDG-style, CLI-namespaced state dir
  // 4. wait GRACE_PERIOD_MS, then re-check the child is still alive before reporting success
}

function run(args) {
  // spawn the same server in the foreground; propagate its exit code directly
}

function stop(args) {
  // shell out to `funshell port <port> --kill` against the port this CLI bound to,
  // then remove the PID file; this is a non-Python CLI, so it shells out to the
  // `funshell` command rather than depending on the `funshell` Python package
}

function status(args) {
  // report PID/port plus this package's own version from package.json
}

module.exports = { start, run, stop, status };
```

`src/static-server.js` and `src/proxy.js` hold the actual serving and reverse-proxy logic described in [runtime-patterns.md](runtime-patterns.md#providing-the-missing-cli-a-self-published-npm-wrapper) — SPA fallback, gzip/conditional requests/`Cache-Control` for the former; hop-by-hop header stripping, body-size limits, and ambiguous-request rejection for the latter. Keep both stateless and independent of `daemonize.js` so they can be exercised standalone in tests.

## Optional Extensions

- Add `install-dev` (dev-only, no argument — clear, rebuild from the working tree, local-install, e.g. via `funbuild install`) and `publish` (prod-only, no argument — build and release, e.g. via `funbuild build`/`funbuild release`) only when the lifecycle script owns those proven commands. Follow `service-release-governance`; never make `start`/`run` publish, install, or fall back to local build output.
- Add `install-prod [version]` (prod-only, optional version — pulls the formal package from the registry/index directly via the ecosystem's package manager, e.g. `pip install "${PKG}${1:+==$1}"`; a no-op if already installed when no version is given, pinned to exactly `$1` when one is) only when the lifecycle script owns the very first install on a host, since the CLI can't install itself before it exists.
- Add `upgrade [version]`, `rollback <version>`, and `uninstall` only as thin passthroughs to the installed CLI's own top-level subcommands (`"${CLI_NAME}" upgrade "$@"`, `"${CLI_NAME}" rollback "$1"`, `"${CLI_NAME}" uninstall`) when the script genuinely needs one consistent entrypoint — the CLI already implements these standalone, which matters on a prod host that may not have this repository checked out at all. Fall back to calling the ecosystem's package manager directly only when the CLI has no such subcommand. Make `uninstall` call `do_stop` first, then invoke the CLI's `uninstall`, never remove a live install.
- Add `all` only when batch operation is required. Implement it as a separate dispatch path with deterministic service order, output labeling, and an explicit fail-fast or collect-errors policy.
- Fall back to a script-managed PID file and service-scoped lock only when the installed CLI cannot daemonize itself (see the fallback in [runtime-patterns.md](runtime-patterns.md#backgrounding-pattern)) — under the primary, CLI-owned-PID model above, concurrent `start`/`stop` safety is the CLI's responsibility, not this script's.
- Extract helpers into `scripts/lib/` only after two or more service scripts share the same tested mechanics.
