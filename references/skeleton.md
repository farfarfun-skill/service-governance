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
  printf 'Usage: %s <action> <service> [dev|prod]\n' "${0##*/}" >&2
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

needs_env() {
  case "$1" in
    start|stop|restart|run) return 0 ;;
    *) return 1 ;;
  esac
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
  local env="${3:-}"
  local script
  local -a args=("${action}")

  script="$(service_script_for "${service}")"
  [[ -f "${script}" ]] || die "missing service script: ${script}"
  needs_env "${action}" && args+=("${env}")
  bash "${script}" "${args[@]}"
}

main() {
  (( $# <= 3 )) || {
    usage
    die "too many arguments"
  }

  local action="${1:-}"
  local service="${2:-}"
  local env="${3:-}"

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

  if needs_env "${action}"; then
    [[ -n "${env}" ]] || env="$(choose dev prod)"
    [[ "${env}" == "dev" || "${env}" == "prod" ]] || {
      usage
      die "invalid environment: ${env}"
    }
  elif [[ -n "${env}" ]]; then
    usage
    die "${action} does not accept an environment"
  fi

  dispatch "${action}" "${service}" "${env}"
}

main "$@"
```

This baseline assumes every listed service supports every listed action. If capabilities differ, add an explicit action-to-service matrix, filter interactive service choices, and reject unsupported combinations before dispatch.

## Per-Service Script

Let each service script own its paths, ports, runtime files, concrete commands, and PID lifecycle:

```bash
#!/usr/bin/env bash
set -euo pipefail

SERVICE_NAME="api"
DEV_PORT=8000
PROD_PORT=8080
STARTUP_GRACE_SECONDS=1
STOP_TIMEOUT_SECONDS=10

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
SCRIPT_PATH="${SCRIPT_DIR}/${BASH_SOURCE[0]##*/}"
ROOT="$(cd "${SCRIPT_DIR}/../.." && pwd)"
SERVICE_ROOT="${ROOT}/services/api"
RUN_DIR="${SERVICE_ROOT}/.run"

readonly SERVICE_NAME DEV_PORT PROD_PORT
readonly STARTUP_GRACE_SECONDS STOP_TIMEOUT_SECONDS
readonly SCRIPT_DIR SCRIPT_PATH ROOT SERVICE_ROOT RUN_DIR

SERVICE_COMMAND=()

usage() {
  printf 'Usage: %s <start|stop|restart|run> <dev|prod>\n' "${0##*/}" >&2
  printf '       %s status\n' "${0##*/}" >&2
}

die() {
  printf 'error: %s\n' "$*" >&2
  exit 2
}

validate_env() {
  [[ "$1" == "dev" || "$1" == "prod" ]]
}

port_for_env() {
  case "$1" in
    dev) printf '%s\n' "${DEV_PORT}" ;;
    prod) printf '%s\n' "${PROD_PORT}" ;;
    *) return 1 ;;
  esac
}

log_file_for_env() {
  printf '%s/%s.%s.log\n' "${RUN_DIR}" "${SERVICE_NAME}" "$1"
}

pid_file_for_env() {
  printf '%s/%s.%s.pid\n' "${RUN_DIR}" "${SERVICE_NAME}" "$1"
}

service_command_for_env() {
  local env="$1"
  local port
  port="$(port_for_env "${env}")"

  # Replace these arrays with commands proven from repository metadata or docs.
  case "${env}" in
    dev) SERVICE_COMMAND=(replace-with-dev-command --port "${port}") ;;
    prod) SERVICE_COMMAND=(replace-with-prod-command --port "${port}") ;;
  esac
}

working_dir_for_env() {
  case "$1" in
    dev) printf '%s\n' "${SERVICE_ROOT}" ;;
    prod) printf '%s\n' "${RUN_DIR}" ;;
    *) return 1 ;;
  esac
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
  local env="$1"
  local working_dir
  service_command_for_env "${env}"
  working_dir="$(working_dir_for_env "${env}")"
  mkdir -p "${working_dir}"
  cd "${working_dir}"
  exec "${SERVICE_COMMAND[@]}"
}

do_start() {
  local env="$1"
  local log_file pid_file pid tmp_pid_file
  log_file="$(log_file_for_env "${env}")"
  pid_file="$(pid_file_for_env "${env}")"

  mkdir -p "${RUN_DIR}"
  if pid="$(read_pid "${pid_file}")" && pid_is_live "${pid}"; then
    die "${SERVICE_NAME} ${env} is already running (pid ${pid})"
  fi
  if [[ -e "${pid_file}" ]]; then
    printf 'warning: removing stale PID file %s\n' "${pid_file}" >&2
    rm -f "${pid_file}"
  fi

  nohup bash "${SCRIPT_PATH}" __run "${env}" \
    </dev/null >>"${log_file}" 2>&1 &
  pid=$!

  tmp_pid_file="${pid_file}.tmp.$$"
  printf '%s\n' "${pid}" >"${tmp_pid_file}"
  mv -f "${tmp_pid_file}" "${pid_file}"

  sleep "${STARTUP_GRACE_SECONDS}"
  if ! pid_is_live "${pid}"; then
    rm -f "${pid_file}"
    printf 'error: %s %s failed to start; inspect %s\n' \
      "${SERVICE_NAME}" "${env}" "${log_file}" >&2
    return 1
  fi

  printf '%s %s started (pid %s, log %s)\n' \
    "${SERVICE_NAME}" "${env}" "${pid}" "${log_file}"
}

do_stop() {
  local env="$1"
  local pid_file pid deadline
  pid_file="$(pid_file_for_env "${env}")"

  if ! pid="$(read_pid "${pid_file}")"; then
    rm -f "${pid_file}"
    printf '%s %s is not running\n' "${SERVICE_NAME}" "${env}"
    return
  fi
  if ! pid_is_live "${pid}"; then
    rm -f "${pid_file}"
    printf '%s %s had a stale PID file\n' "${SERVICE_NAME}" "${env}"
    return
  fi

  kill -TERM "${pid}"
  deadline=$((SECONDS + STOP_TIMEOUT_SECONDS))
  while pid_is_live "${pid}"; do
    if (( SECONDS >= deadline )); then
      printf 'error: %s %s did not stop after %ss (pid %s)\n' \
        "${SERVICE_NAME}" "${env}" "${STOP_TIMEOUT_SECONDS}" "${pid}" >&2
      return 1
    fi
    sleep 0.2
  done

  rm -f "${pid_file}"
  printf '%s %s stopped\n' "${SERVICE_NAME}" "${env}"
}

do_restart() {
  local env="$1"
  do_stop "${env}"
  do_start "${env}"
}

do_status_env() {
  local env="$1"
  local pid_file pid port
  pid_file="$(pid_file_for_env "${env}")"
  port="$(port_for_env "${env}")"

  if pid="$(read_pid "${pid_file}")" && pid_is_live "${pid}"; then
    printf '%s %s: running (pid %s, configured port %s)\n' \
      "${SERVICE_NAME}" "${env}" "${pid}" "${port}"
  elif [[ -e "${pid_file}" ]]; then
    printf '%s %s: stale PID file (%s)\n' \
      "${SERVICE_NAME}" "${env}" "${pid_file}"
  else
    printf '%s %s: stopped (configured port %s)\n' \
      "${SERVICE_NAME}" "${env}" "${port}"
  fi
}

do_status() {
  do_status_env dev
  do_status_env prod
}

main() {
  local action="${1:-}"
  local env="${2:-}"

  case "${action}" in
    __run)
      (( $# == 2 )) || die "invalid internal invocation"
      validate_env "${env}" || die "invalid environment: ${env}"
      do_run "${env}"
      ;;
    start|stop|restart|run)
      (( $# == 2 )) || {
        usage
        die "${action} requires exactly one environment"
      }
      validate_env "${env}" || die "invalid environment: ${env}"
      "do_${action}" "${env}"
      ;;
    status)
      (( $# == 1 )) || {
        usage
        die "status does not accept an environment"
      }
      do_status
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

- Add a separate `publish` or `install` action only when the lifecycle script owns that proven release command. Follow `service-release-governance`; never make `start prod` publish, install, or fall back to local build output.
- Add `all` only when batch operation is required. Implement it as a separate dispatch path with deterministic service order, environment handling, output labeling, and an explicit fail-fast or collect-errors policy.
- Add service-and-environment-scoped locking when concurrent lifecycle calls are plausible.
- Extract helpers into `scripts/lib/` only after two or more service scripts share the same tested mechanics.
