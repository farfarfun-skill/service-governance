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
STARTUP_GRACE_SECONDS=1
STOP_TIMEOUT_SECONDS=10

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
SCRIPT_PATH="${SCRIPT_DIR}/${BASH_SOURCE[0]##*/}"
ROOT="$(cd "${SCRIPT_DIR}/../.." && pwd)"
SERVICE_ROOT="${ROOT}/services/api"
RUN_DIR="${SERVICE_ROOT}/.run"
LOG_FILE="${RUN_DIR}/${SERVICE_NAME}.log"
PID_FILE="${RUN_DIR}/${SERVICE_NAME}.pid"

readonly SERVICE_NAME CLI_NAME PORT CONFIG_PATH
readonly STARTUP_GRACE_SECONDS STOP_TIMEOUT_SECONDS
readonly SCRIPT_DIR SCRIPT_PATH ROOT SERVICE_ROOT RUN_DIR LOG_FILE PID_FILE

SERVICE_COMMAND=()

usage() {
  printf 'Usage: %s <start|stop|restart|run|status>\n' "${0##*/}" >&2
}

die() {
  printf 'error: %s\n' "$*" >&2
  exit 2
}

service_command() {
  # Both a locally-built install and a registry-pinned install expose the
  # same CLI; do not branch this on dev/prod.
  SERVICE_COMMAND=("${CLI_NAME}" server start --port "${PORT}")
  [[ -n "${CONFIG_PATH}" ]] && SERVICE_COMMAND+=(--config "${CONFIG_PATH}")
}

installed_version() {
  "${CLI_NAME}" --version 2>/dev/null || printf 'unknown\n'
}

read_pid() {
  local pid_file="$1"
  local pid
  [[ -f "${pid_file}" ]] || return 1
  IFS= read -r pid <"${pid_file}" || return 1
  [[ "${pid}" =~ ^[0-9]+$ ]] && (( pid > 1 )) || return 1
  printf '%s\n' "${pid}"
}

pid_is_live() {
  kill -0 "$1" 2>/dev/null
}

do_run() {
  service_command
  mkdir -p "${SERVICE_ROOT}"
  cd "${SERVICE_ROOT}"
  exec "${SERVICE_COMMAND[@]}"
}

do_start() {
  local pid tmp_pid_file

  mkdir -p "${RUN_DIR}"
  if pid="$(read_pid "${PID_FILE}")" && pid_is_live "${pid}"; then
    die "${SERVICE_NAME} is already running (pid ${pid})"
  fi
  if [[ -e "${PID_FILE}" ]]; then
    printf 'warning: removing stale PID file %s\n' "${PID_FILE}" >&2
    rm -f "${PID_FILE}"
  fi

  nohup bash "${SCRIPT_PATH}" __run \
    </dev/null >>"${LOG_FILE}" 2>&1 &
  pid=$!

  tmp_pid_file="${PID_FILE}.tmp.$$"
  printf '%s\n' "${pid}" >"${tmp_pid_file}"
  mv -f "${tmp_pid_file}" "${PID_FILE}"

  sleep "${STARTUP_GRACE_SECONDS}"
  if ! pid_is_live "${pid}"; then
    rm -f "${PID_FILE}"
    printf 'error: %s failed to start; inspect %s\n' \
      "${SERVICE_NAME}" "${LOG_FILE}" >&2
    return 1
  fi

  printf '%s started (pid %s, log %s)\n' "${SERVICE_NAME}" "${pid}" "${LOG_FILE}"
}

do_stop() {
  local pid deadline

  if ! pid="$(read_pid "${PID_FILE}")"; then
    rm -f "${PID_FILE}"
    printf '%s is not running\n' "${SERVICE_NAME}"
    return
  fi
  if ! pid_is_live "${pid}"; then
    rm -f "${PID_FILE}"
    printf '%s had a stale PID file\n' "${SERVICE_NAME}"
    return
  fi

  kill -TERM "${pid}"
  deadline=$((SECONDS + STOP_TIMEOUT_SECONDS))
  while pid_is_live "${pid}"; do
    if (( SECONDS >= deadline )); then
      printf 'error: %s did not stop after %ss (pid %s)\n' \
        "${SERVICE_NAME}" "${STOP_TIMEOUT_SECONDS}" "${pid}" >&2
      return 1
    fi
    sleep 0.2
  done

  rm -f "${PID_FILE}"
  printf '%s stopped\n' "${SERVICE_NAME}"
}

do_restart() {
  do_stop
  do_start
}

do_status() {
  local pid

  if pid="$(read_pid "${PID_FILE}")" && pid_is_live "${pid}"; then
    printf '%s: running (pid %s, configured port %s, version %s)\n' \
      "${SERVICE_NAME}" "${pid}" "${PORT}" "$(installed_version)"
  elif [[ -e "${PID_FILE}" ]]; then
    printf '%s: stale PID file (%s)\n' "${SERVICE_NAME}" "${PID_FILE}"
  else
    printf '%s: stopped (configured port %s, version %s)\n' \
      "${SERVICE_NAME}" "${PORT}" "$(installed_version)"
  fi
}

main() {
  local action="${1:-}"

  case "${action}" in
    __run)
      (( $# == 1 )) || die "invalid internal invocation"
      do_run
      ;;
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

The generic `pid_is_live` check proves only liveness. Extend it with a repository-specific identity check when a stable executable, command marker, or runtime metadata source is available. Do not substitute a port-owner lookup as process identity.

For a true single-service repository, keep the same validation and lifecycle boundaries in `scripts/setup.sh` and omit the dispatcher layer.

## Optional Extensions

- Add `install` (dev-only, no argument — clear, rebuild from the working tree, local-install, e.g. via `funbuild install`) and `publish` (prod-only, no argument — build and release, e.g. via `funbuild build`/`funbuild release`) only when the lifecycle script owns those proven commands. Follow `service-release-governance`; never make `start`/`run` publish, install, or fall back to local build output.
- Add `upgrade`, `rollback <version>`, and `uninstall` only as thin passthroughs to the installed CLI's own top-level subcommands (`"${CLI_NAME}" upgrade`, `"${CLI_NAME}" rollback "$1"`, `"${CLI_NAME}" uninstall`) when the script genuinely needs one consistent entrypoint — the CLI already implements these standalone, which matters on a prod host that may not have this repository checked out at all. Make `uninstall` call `do_stop` first, then invoke the CLI's `uninstall`, never remove a live install.
- Add `all` only when batch operation is required. Implement it as a separate dispatch path with deterministic service order, output labeling, and an explicit fail-fast or collect-errors policy.
- Add service-scoped locking when concurrent lifecycle calls are plausible.
- Extract helpers into `scripts/lib/` only after two or more service scripts share the same tested mechanics.
