# Rules And Review Checklist

Read this file when you need the full invariant set or want to review an existing service script layout.

- [Invariants](#invariants)
- [Review Checklist](#review-checklist)
- [Common Mistakes](#common-mistakes)
- [Anti-Patterns](#anti-patterns)

## Invariants

### Entry Point

- Use `scripts/setup.sh` as the single user-facing lifecycle entrypoint.
- Allow helper files under `scripts/`, but route external operations through `setup.sh`.
- If the repository has multiple services, prefer `setup.sh` as a dispatcher and move service-specific logic into separate scripts.

### Service Layout

- Single-service repositories may keep all lifecycle logic in `scripts/setup.sh`.
- Split into per-service scripts when any of these is true:
  - there are two or more long-running services
  - frontend and backend use clearly different runtime models
  - services have different ports, build outputs, or production entrypoints
  - one script has started to accumulate large service-specific branching
- Keep shared helpers in `scripts/lib/` only when duplication is real.
- Put only shared mechanics in `scripts/lib/`, such as logging helpers, PID handling, signal handling, and port checks.
- Do not put service-specific start commands, build commands, or repository-path assumptions into `scripts/lib/`.

### Command Semantics

None of `start`, `stop`, `restart`, `run`, or `status` take a `dev`/`prod` argument. They act on whichever package is currently installed on the host — a host only ever has one active install, and whoever installed it already knows whether it came from a local build or the registry. Only `install` and `publish` are inherently environment-specific, and neither needs an explicit argument because the action name already says which one it is.

- `start`: run in the background and detach from the current shell session. Invokes the installed package's CLI, `<cli> server start [--config <path>]`.
- `run`: same command as `start`, but stay in the foreground in the current terminal (`exec`, no backgrounding).
- `stop`: stop the target process for the selected service.
- `restart`: implement as `stop` + `start` or equivalent explicit logic.
- `status`: report the service's state without prompting, including the installed package's version (e.g. via `pip show`/`npm ls -g`/the CLI's own `--version`) alongside PID/port.
- `publish`: build and upload a new formal package; prod-only, always the release flavor. Call `funbuild build` (alias `funbuild release`) when the project uses this convention — it already covers version bump, build, publish, push, and tag for npm, Python, and hybrid repos.
- `install`: dev-only, always the local-build flavor — clear the previous local build/install, rebuild from the current working tree, and install the built package locally (`funbuild install` when available). There is no `prod` counterpart in the lifecycle script: a prod host installs the exact formal version directly via the ecosystem's package manager or the installed CLI's own `upgrade`/`rollback` subcommand, per `service-release-governance`, not through this script.
- `upgrade`/`rollback <version>`/`uninstall`: primarily the installed CLI's own top-level subcommands (`<cli> upgrade`, `<cli> rollback <version>`, `<cli> uninstall`), not something to hand-roll in Bash. Add a passthrough in the lifecycle script only when it genuinely needs one consistent entrypoint for all of these. `uninstall` must stop the running service first, then remove the package — never remove it out from under a live process. There is no separate `install`-for-rollback path; rolling back means `rollback <version>`.
- `all`: optional aggregate target, not a required baseline feature.

Keep `publish`, `install`, and any `upgrade`/`rollback`/`uninstall` passthrough separate from `start`/`run`; a start must not silently build, publish, install, or change the installed version. Expose these only when the Bash lifecycle script actually owns that operation. Otherwise leave them to the installed CLI and release automation, and treat the installed package as a prerequisite for `start`/`run`.

Do not expose commands that the service does not actually support. Apply [Service Release Governance](../../service-release-governance/SKILL.md) to every production publish, install, and start path.

Use `exec` for the final foreground command so signals and exit codes reach the service directly. Ensure the managed production command does not self-daemonize; if it does, use its native PID mechanism or the repository's process supervisor.

### Selection Order

- Treat the target service as explicit state when the repository contains multiple long-running services.
- Preserve the parse order `action -> service`. No baseline action takes an environment argument, so there is no further `-> env` step to resolve.
- Single-service repositories may omit the service argument.
- If aggregate mode is supported, treat `all` as an explicit pseudo-service rather than an implicit default.

### Runtime Files

- Create `.run/` under the service root before writing runtime files.
- If one dispatcher manages several services, either:
  - write runtime files into each service root's `.run/`, or
  - use a repo-level `.run/` with service-qualified filenames such as `api.pid`
- Keep filenames specific enough to avoid collisions across services. No environment suffix is needed — a host only ever has one active install per service.
- Avoid `/tmp` unless the repository explicitly requires it.

### Process Ownership

- Before `start`, reject a live PID file and remove or report a stale one explicitly.
- After backgrounding, write the captured PID atomically and verify that it survives a short startup grace period.
- Redirect stdin, stdout, and stderr for background starts so the process is detached from the terminal.
- Before `stop`, require a numeric PID, check that it is live, and verify service identity when the repository offers a reliable command or metadata check.
- Send `TERM`, wait for a bounded interval, and remove the PID file only after the process exits. Use `KILL` only when repository policy explicitly permits forced shutdown.
- Never discover a process by port and then kill it. A listener can belong to an unrelated process.
- Serialize lifecycle changes with a service-scoped lock when concurrent invocations are realistic.

### Interaction

- Preserve the repository's established prompt tool when one exists.
- When using `gum`, keep fully specified CLI calls independent of it and emit a clear usage error if an interactive choice is needed but `gum` is unavailable.
- Never install an interactive dependency from a lifecycle script.

### Environment Identity

- Do not add a `dev`/`prod` argument to `start`, `stop`, `restart`, `run`, or `status`. Environment identity lives entirely in what got installed (`install` vs. a registry install/`upgrade`/`rollback`), never in a runtime argument.
- Preserve the order: action first, service second when needed. No environment step follows.
- Support direct CLI usage without prompting when action and service are already present.
- Reject invalid and extra args instead of silently discarding them or replacing them with a prompt.
- Make `status` report the installed package's identity (name, version) and runtime state without opening a menu.

### Aggregate Operations

- Do not add `all` unless the repository actually needs batch start, stop, restart, or status behavior.
- If `all` is supported, define a deterministic service order and document it in the script.
- Decide failure handling explicitly: fail fast, or continue and report per-service failures at the end.
- Keep aggregate output readable by prefixing lines with the service name or clearly separating sections.
- Do not let `all` change the semantics of single-service commands.

### Port Configuration

- Define ports in the top configuration block.
- For a single service, this shape is fine:

```bash
PORT=8080
```

- For multiple services, use service-scoped names such as:

```bash
API_PORT=8080
WEB_PORT=4173
```

- No environment suffix is needed — a host only ever runs one active install, so one port per service is enough. If a repository genuinely needs to run a local build and a registry install side by side on one host for comparison, treat that as a repository-specific extension, not the baseline.
- Reuse only these variables, or readonly variables derived from them, throughout startup, health checks, and status output.
- Do not hardcode port numbers inside branch logic.

### Start Rules

- `start`/`run` use exactly one command shape regardless of what's installed: `<installed-cli> server start [--config <path>] [--port ...]`. Whether the underlying package came from a local build or the registry-pinned version is decided entirely at install time, not by an argument to `start`/`run`.
- `--config` is optional. When passed, it typically points `install`-time dev usage at a repo-tracked file such as `config/dev.toml`; when omitted, the CLI resolves its own hardcoded default (`${XDG_CONFIG_HOME:-~/.config}/<org>/<cli-name>/config.toml`), which is normally sufficient for a prod host.
- Ad hoc hot-reload tooling (`npm run dev`, `vite dev`, `next dev`, `uv run --reload`) is fine as an unmanaged workflow while actively editing code, but is never what the lifecycle script's `start`/`run` itself invokes — that action must go through the CLI, backed by a package actually installed from the current source.
- For Python services, run the installed CLI entrypoint (`<cli> server start`), never a raw module path, script, editable install, `PYTHONPATH` override, or an implicit `uv run` workspace resolution.
- For frontend services, run the installed CLI/production server from the exact installed package, not the checkout's `dist/`/`build/` directory run directly.
- In mixed frontend/backend repositories, choose the install and start commands independently for each service.
- Fail `start`/`run` when the required installed package is absent; never fall back to source or a stale install.

## Review Checklist

- Does `setup.sh` own the user-facing lifecycle interface?
- If the repo has multiple services, is service selection explicit and deterministic?
- If the repo has multiple services, is the split threshold met, or is one large script still carrying too much service-specific logic?
- Does `start` fully detach and write runtime state into `.run/`?
- Does `run` stay in the foreground?
- Does `run` preserve service signals and exit status with `exec`?
- Do `start`, `stop`, `restart`, `run`, and `status` correctly take no `dev`/`prod` argument, leaving environment identity entirely to what's installed?
- Is the menu flow action-first and service-second when needed?
- If `all` exists, are service order, output format, and failure policy explicit?
- Are ports centralized at the top of the relevant script?
- Are status and health checks derived from the same service-specific port configuration?
- Are duplicate starts, stale PID files, failed starts, and graceful stop timeouts handled explicitly?
- Does `stop` avoid signaling a PID based only on a port match?
- Does `start`/`run` use the exact package actually installed on the host — a local build after `install`, or the formal package installed from the intended repository — for each affected service?
- Does `status` report the installed package's version, not just PID/port?
- Does `uninstall` stop the running service first and only then remove the package?
- Are unsupported commands hidden or rejected clearly?

## Common Mistakes

- Keeping all frontend and backend lifecycle logic in one giant `setup.sh` after the repository became multi-service.
- Adding `all` without defining execution order or what happens after one service fails.
- Adding a `dev`/`prod` argument to `start`, `stop`, `restart`, or `run` when the installed package already fixes which one is running.
- Guessing which service the user meant when both frontend and backend exist.
- Managing multiple services while still using ambiguous runtime file names when a service-scoped name such as `.run/api.pid` would do.
- Moving service-specific startup logic into `scripts/lib/` and turning the shared library into another monolith.
- Implementing `start` in a way that dies when the terminal closes.
- Letting `run` silently fork into the background.
- Capturing the PID of a wrapper shell that does not `exec` the actual service.
- Trusting a stale PID file or killing whichever process happens to own the configured port.
- Building commands as strings and executing them with `eval` or an unnecessary `bash -c`.
- Leaving ports scattered across functions.
- Pointing a running service at a locally built package when the host is meant to run the registry-installed formal version, or vice versa.
- Writing logs without service-specific naming where collisions across services are possible.
- Hand-rolling `upgrade`/`rollback`/`uninstall` in Bash instead of delegating to the installed CLI's own subcommands.
- Removing an installed package in `uninstall` without stopping the running service first.
- Shipping placeholder lifecycle commands that do nothing.

## Anti-Patterns

- A single `setup.sh` that handles `api`, `web`, `admin`, and `worker` entirely through nested `case` blocks.
- Repeated `cd` hopping across several directories to infer which service is being operated.
- One shared PID file or one shared log file for several services.
- Hiding aggregate behavior behind normal commands so `start` sometimes means one service and sometimes means all services.
- A `scripts/lib/` folder that knows concrete package names, ports, or build output paths for individual services.
