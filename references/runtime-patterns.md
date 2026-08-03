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

Prefer these production entrypoints, in order of confidence:

1. A console script from the exact formal package installed into the production environment
2. `python -m package` using that production environment, with the working directory outside the source checkout

Avoid these as the default `start prod` command:

- `python scripts/foo.py`
- `python app.py` from an ad hoc working directory
- `uv run <command>` when it can resolve the current workspace or an editable install
- Dev-only reload servers used as production daemons

Typical split:

- `run dev`: local dev server, reloader, or `uv run ... --reload`
- `publish`: configured `nltbuild build` or the existing Python release command
- production install: exact package version from the repository chosen by `service-release-governance`
- `start prod`: installed CLI or module entrypoint without dev-only reload flags or source-path overrides

In multi-service repositories:

- keep Python service commands scoped to that service's root, virtualenv tooling, and ports
- do not reuse backend defaults for unrelated frontend services

## Frontend Pattern

Prefer these production entrypoints:

1. A documented production server command from the exact package installed from the selected registry
2. An SSR server or static asset command whose `build/` or `dist/` belongs to that installed release, outside the source checkout

Avoid these as `start prod` defaults:

- `vite dev`
- `next dev`
- Any hot-reload or watch command

Typical split:

- `run dev`: `npm run dev`, `pnpm dev`, or framework-equivalent local dev server
- `publish`: build and upload the formal package through the configured registry
- production install: exact package version from that registry into a clean release location
- `start prod`: SSR server or static asset command against the installed release output

In multi-service repositories:

- keep development commands scoped to the service root and production commands scoped to the installed release root
- do not assume the frontend and backend share the same `publish` or `start prod` pipeline

## Backgrounding Pattern

Build commands as Bash arrays so arguments remain distinct:

```bash
command=(uv run api --port "${port}")
nohup "${command[@]}" </dev/null >>"${log_file}" 2>&1 &
pid=$!
```

If background start reuses the foreground implementation, invoke an internal script action and make the foreground path end in `exec`:

```bash
nohup bash "${SCRIPT_PATH}" __run "${env}" </dev/null >>"${log_file}" 2>&1 &
pid=$!

do_run() {
  service_command_for_env "$1"
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
- In multi-service scripts, ensure the PID and log file path is unique for both service and environment.
- Do not use `eval` or a command string. Use `bash -c` only when shell syntax is genuinely required and parameters can be passed safely.
- If the command daemonizes itself or leaves an unmanaged process tree, use the runtime's native PID mechanism or an existing process supervisor.

## Sanity Checks

- Does `status` inspect the same service-specific and environment-specific port that `start` used?
- Does `stop` target the PID file written by the chosen background start path?
- Does `stop` send `TERM`, wait for exit, and avoid deleting state while the process is still live?
- Can a stale or reused PID cause the script to signal an unrelated process?
- Does each service's `prod` path avoid reload, watch, and development-only flags?
- Does each `prod` command resolve only the pinned repository-installed package, never the source checkout or local build output?
- Does the script rely on the repository root or activated environment in a way that must be made explicit?
