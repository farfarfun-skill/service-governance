# Runtime Patterns

Read this file when `scripts/setup.sh` or a per-service script needs concrete runtime commands or when `prod` behavior is easy to get wrong.

- [Choose The Right Pattern](#choose-the-right-pattern)
- [Python Pattern](#python-pattern)
- [Frontend Pattern](#frontend-pattern)
- [Backgrounding Pattern](#backgrounding-pattern)
- [Sanity Checks](#sanity-checks)

## Choose The Right Pattern

- If the repository has `pyproject.toml`, `requirements.txt`, or a Python package entrypoint, use the Python pattern for that service.
- If the repository has `package.json`, `vite.config.*`, `next.config.*`, or a build output directory such as `dist/`, use the frontend pattern for that service.
- If the repository has both backend and frontend runtimes, apply the matching pattern per service instead of collapsing them into one command model.
- If neither pattern fits, preserve repository-specific conventions and still honor the core rules from [rules.md](rules.md).

## Python Pattern

`start`/`run` always use the installed console-script entrypoint, regardless of whether the install came from a local build or the registry. Prefer, in order of confidence:

1. A console script from whichever package is currently installed (a locally built copy after `install`, or the exact formal package after a registry install)
2. `python -m package` using that same environment, with the working directory outside the source checkout

Avoid these as the `start`/`run` command, in either case:

- `python scripts/foo.py`
- `python app.py` from an ad hoc working directory
- `uv run <command>` when it can resolve the current workspace or an editable install
- Dev-only reload servers used as the lifecycle script's daemon

Typical split — only `install` and `publish` are environment-specific; `start`/`run`/`stop`/`status` are identical either way:

- ad hoc dev iteration (outside the lifecycle script): local dev server, reloader, or `uv run ... --reload`
- `install` (dev-only): clear previous build/install, rebuild, force-reinstall the wheel locally (`funbuild install` when available)
- `publish` (prod-only): `funbuild build` (alias `funbuild release`) or the existing Python release command
- production install (done outside the repo's `setup.sh`, on the prod host): exact package version from the repository chosen by `service-release-governance`, via `pip`/`uv pip` or the installed CLI's own `upgrade`/`rollback` subcommand
- `start`/`run`: the installed CLI or module entrypoint, no dev-only reload flags or source-path overrides, no `dev`/`prod` argument
- `status`: PID/port plus the installed package's version

In multi-service repositories:

- keep Python service commands scoped to that service's root, virtualenv tooling, and ports
- do not reuse backend defaults for unrelated frontend services

## Frontend Pattern

`start`/`run` always use the installed production server/CLI, regardless of whether the install came from a local build or the registry. Prefer:

1. A documented production server command from whichever package is currently installed
2. An SSR server or static asset command whose `build/` or `dist/` belongs to that installed release, outside the source checkout

Avoid these as the `start`/`run` command:

- `vite dev`
- `next dev`
- Any hot-reload or watch command

Typical split — only `install` and `publish` are environment-specific; `start`/`run`/`stop`/`status` are identical either way:

- ad hoc dev iteration (outside the lifecycle script): `npm run dev`, `pnpm dev`, or framework-equivalent local dev server
- `install` (dev-only): clear previous build/install, rebuild (`npm run build`), install the built package locally (`funbuild install` when available)
- `publish` (prod-only): build and upload the formal package through the configured registry
- production install (done outside the repo's `setup.sh`, on the prod host): exact package version from that registry, via `npm install -g`/`npm ci` or the installed CLI's own `upgrade`/`rollback` subcommand
- `start`/`run`: SSR server or static asset command against whichever installed release is currently on the host, no `dev`/`prod` argument
- `status`: PID/port plus the installed package's version

In multi-service repositories:

- keep ad hoc dev iteration scoped to the service root and `start`/`run` scoped to the installed release root
- do not assume the frontend and backend share the same `publish` pipeline

## Backgrounding Pattern

Build commands as Bash arrays so arguments remain distinct:

```bash
command=("${CLI_NAME}" server start --port "${port}")
nohup "${command[@]}" </dev/null >>"${log_file}" 2>&1 &
pid=$!
```

If background start reuses the foreground implementation, invoke an internal script action and make the foreground path end in `exec`:

```bash
nohup bash "${SCRIPT_PATH}" __run </dev/null >>"${log_file}" 2>&1 &
pid=$!

do_run() {
  service_command
  cd "${SERVICE_ROOT}"
  exec "${SERVICE_COMMAND[@]}"
}
```

Whichever style you choose:

- Capture `$!` immediately and write it to the PID file atomically.
- Refuse a duplicate start when the existing PID is live; handle stale or invalid PID files explicitly.
- Verify that the child survives a short startup grace period before reporting success.
- Write logs into `.run/`.
- Redirect stdin from `/dev/null` and do not let `start` depend on the terminal staying open.
- In multi-service scripts, ensure the PID and log file path is unique per service (e.g. `.run/api.pid`) — no environment suffix is needed since a host only runs one active install per service.
- Do not use `eval` or a command string. Use `bash -c` only when shell syntax is genuinely required and parameters can be passed safely.
- If the command daemonizes itself or leaves an unmanaged process tree, use the runtime's native PID mechanism or an existing process supervisor.

## Sanity Checks

- Does `status` inspect the same service-specific port that `start` used, and report the installed package's version alongside it?
- Does `stop` target the PID file written by the chosen background start path?
- Does `stop` send `TERM`, wait for exit, and avoid deleting state while the process is still live?
- Can a stale or reused PID cause the script to signal an unrelated process?
- Does `start`/`run` avoid reload, watch, and development-only flags regardless of which package is installed?
- Does `start`/`run` resolve only the currently installed package — a fresh local build after `install`, or the pinned registry install — never the source checkout or a stale build output?
- Does the script correctly avoid taking a `dev`/`prod` argument on `start`/`stop`/`restart`/`run`/`status`?
- Does `uninstall`, if the script owns it, stop the service before removing the package?
- Does the script rely on the repository root or activated environment in a way that must be made explicit?
