# Starter Skeleton

Use this file only when drafting or heavily rewriting service lifecycle scripts. Adapt service names, actions, commands, paths, ports, and process-identity checks to the repository; do not copy placeholders literally.

- [Dispatcher](#dispatcher)
- [Per-Service Script](#per-service-script)
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

This baseline assumes every listed service supports every listed action, and that none of them take a `dev`/`prod` argument — a host only ever runs one active install per service, so `start`/`stop`/`restart`/`run`/`status` act on whatever is currently installed. If capabilities differ per service, add an explicit action-to-service matrix, filter interactive service choices, and reject unsupported combinations before dispatch. Add `install`/`publish` per [Optional Extensions](#optional-extensions) only when `setup.sh` owns those steps; neither takes an environment argument either, since the action name already fixes which one it is.

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
  [[ -n "${CONFIG_PATH}" ]] && CLI_ARGS+=(--config "${CONFIG_PATH}")
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

`do_start`/`do_run`/`do_stop`/`do_status` are thin delegations: the installed CLI backgrounds itself (`server start`), stays foregrounded when asked (`server run`), and owns the PID file that `server stop`/`server status` read — this script does not `nohup` anything, write a PID file, or poll for liveness itself. If a given repository's CLI genuinely cannot daemonize itself, fall back to the `nohup`-and-PID-file pattern in [runtime-patterns.md](runtime-patterns.md#backgrounding-pattern) instead of this primary shape; in that fallback, extend the liveness check with a repository-specific identity check when a stable executable, command marker, or runtime metadata source is available, and never substitute a port-owner lookup as process identity.

For a true single-service repository, keep the same validation and lifecycle boundaries in `scripts/setup.sh` and omit the dispatcher layer.

## Optional Extensions

- Add `install` (dev-only, no argument — clear, rebuild from the working tree, local-install, e.g. via `funbuild install`) and `publish` (prod-only, no argument — build and release, e.g. via `funbuild build`/`funbuild release`) only when the lifecycle script owns those proven commands. Follow `service-release-governance`; never make `start`/`run` publish, install, or fall back to local build output.
- Add `upgrade`, `rollback <version>`, and `uninstall` only as thin passthroughs to the installed CLI's own top-level subcommands (`"${CLI_NAME}" upgrade`, `"${CLI_NAME}" rollback "$1"`, `"${CLI_NAME}" uninstall`) when the script genuinely needs one consistent entrypoint — the CLI already implements these standalone, which matters on a prod host that may not have this repository checked out at all. Make `uninstall` call `do_stop` first, then invoke the CLI's `uninstall`, never remove a live install.
- Add `all` only when batch operation is required. Implement it as a separate dispatch path with deterministic service order, output labeling, and an explicit fail-fast or collect-errors policy.
- Fall back to a script-managed PID file and service-scoped lock only when the installed CLI cannot daemonize itself (see the fallback in [runtime-patterns.md](runtime-patterns.md#backgrounding-pattern)) — under the primary, CLI-owned-PID model above, concurrent `start`/`stop` safety is the CLI's responsibility, not this script's.
- Extract helpers into `scripts/lib/` only after two or more service scripts share the same tested mechanics.
