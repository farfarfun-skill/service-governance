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

- `start`: run in the background and detach from the current shell session.
- `run`: stay in the foreground in the current terminal.
- `stop`: stop the target process for the selected service and environment.
- `restart`: implement as `stop` + `start` or equivalent explicit logic.
- `status`: report each configured environment without prompting; accept an environment filter only when the repository defines one.
- `publish`: build and upload a new formal package; call `nltbuild build` when the project uses this convention.
- `install`: install an exact formal version from the repository selected by `service-release-governance`.
- `all`: optional aggregate target, not a required baseline feature.

Keep `publish` and `install` separate from `start prod`; a start must not silently build, publish, or change the installed version. Expose `publish` or `install` only when the Bash lifecycle script actually owns that operation. Otherwise leave them to release automation and treat the installed package as a production prerequisite.

Do not expose commands that the service does not actually support. Apply [Service Release Governance](../../service-release-governance/SKILL.md) to every production publish, install, and start path.

Use `exec` for the final foreground command so signals and exit codes reach the service directly. Ensure the managed production command does not self-daemonize; if it does, use its native PID mechanism or the repository's process supervisor.

### Selection Order

- Treat the target service as explicit state when the repository contains multiple long-running services.
- Preserve the parse order `action -> service -> env` for environment-bound commands.
- Single-service repositories may omit the service argument.
- If aggregate mode is supported, treat `all` as an explicit pseudo-service rather than an implicit default.

### Runtime Files

- Create `.run/` under the service root before writing runtime files.
- If one dispatcher manages several services, either:
  - write runtime files into each service root's `.run/`, or
  - use a repo-level `.run/` with service-qualified filenames such as `api.dev.pid`
- Keep filenames specific enough to avoid collisions across both service and environment.
- Avoid `/tmp` unless the repository explicitly requires it.

### Process Ownership

- Before `start`, reject a live PID file and remove or report a stale one explicitly.
- After backgrounding, write the captured PID atomically and verify that it survives a short startup grace period.
- Redirect stdin, stdout, and stderr for background starts so the process is detached from the terminal.
- Before `stop`, require a numeric PID, check that it is live, and verify service identity when the repository offers a reliable command or metadata check.
- Send `TERM`, wait for a bounded interval, and remove the PID file only after the process exits. Use `KILL` only when repository policy explicitly permits forced shutdown.
- Never discover a process by port and then kill it. A listener can belong to an unrelated process.
- Serialize lifecycle changes with a service-and-environment-scoped lock when concurrent invocations are realistic.

### Interaction

- Preserve the repository's established prompt tool when one exists.
- When using `gum`, keep fully specified CLI calls independent of it and emit a clear usage error if an interactive choice is needed but `gum` is unavailable.
- Never install an interactive dependency from a lifecycle script.

### Environment Selection

- Require `dev` or `prod` selection for `start`, `stop`, `restart`, and `run`.
- Preserve the order: action first, service second when needed, environment last.
- Support direct CLI usage without prompting when action, service, and env are already present.
- Reject invalid and extra args instead of silently discarding them or replacing them with a prompt.
- Make unfiltered `status` report all configured environments without opening a menu.

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
DEV_PORT=3000
PROD_PORT=8080
```

- For multiple services, use service-scoped names such as:

```bash
API_DEV_PORT=8000
API_PROD_PORT=8080
WEB_DEV_PORT=3000
WEB_PROD_PORT=4173
```

- Reuse only these variables, or readonly variables derived from them, throughout startup, health checks, and status output.
- Do not hardcode port numbers inside branch logic.

### Production Start Rules

- Allow development commands to run current source directly.
- For Python services, run an installed CLI entrypoint or module from the production environment; do not let the working tree, an editable install, `PYTHONPATH`, or an implicit `uv run` workspace resolution supply production code.
- For frontend services, run the production server or static assets from the exact repository-installed package or release directory, not the checkout's `dist/` or `build/` directory.
- In mixed frontend/backend repositories, choose the production command independently for each service.
- Do not use `vite dev`, `next dev`, or similar development servers as the production default.
- Keep `dev` and `prod` start commands intentionally separate when they differ.
- Fail `start prod` and `run prod` when the required installed version is absent or differs from the pinned release; never fall back to source.

## Review Checklist

- Does `setup.sh` own the user-facing lifecycle interface?
- If the repo has multiple services, is service selection explicit and deterministic?
- If the repo has multiple services, is the split threshold met, or is one large script still carrying too much service-specific logic?
- Does `start` fully detach and write runtime state into `.run/`?
- Does `run` stay in the foreground?
- Does `run` preserve service signals and exit status with `exec`?
- Are `start`, `stop`, `restart`, and `run` always environment-specific?
- Is the menu flow action-first, service-second when needed, and environment-last?
- If `all` exists, are service order, output format, and failure policy explicit?
- Are ports centralized at the top of the relevant script?
- Are status and health checks derived from the same service-specific and environment-specific port configuration?
- Are duplicate starts, stale PID files, failed starts, and graceful stop timeouts handled explicitly?
- Does `stop` avoid signaling a PID based only on a port match?
- Does `start prod` use the exact formal package installed from the intended repository for each affected service?
- Are unsupported commands hidden or rejected clearly?

## Common Mistakes

- Keeping all frontend and backend lifecycle logic in one giant `setup.sh` after the repository became multi-service.
- Adding `all` without defining execution order or what happens after one service fails.
- Combining `start dev`, `start prod`, `run dev`, and `run prod` into one flat menu.
- Guessing which service the user meant when both frontend and backend exist.
- Managing multiple services while still using single-service runtime file names such as `.run/app.dev.pid`.
- Moving service-specific startup logic into `scripts/lib/` and turning the shared library into another monolith.
- Implementing `start` in a way that dies when the terminal closes.
- Letting `run` silently fork into the background.
- Capturing the PID of a wrapper shell that does not `exec` the actual service.
- Trusting a stale PID file or killing whichever process happens to own the configured port.
- Building commands as strings and executing them with `eval` or an unnecessary `bash -c`.
- Leaving ports scattered across functions.
- Pointing `prod` to a development-only command.
- Writing logs without service and environment suffixes where collisions are possible.
- Shipping placeholder lifecycle commands that do nothing.

## Anti-Patterns

- A single `setup.sh` that handles `api`, `web`, `admin`, and `worker` entirely through nested `case` blocks.
- Repeated `cd` hopping across several directories to infer which service is being operated.
- One shared PID file or one shared log file for several services.
- Hiding aggregate behavior behind normal commands so `start dev` sometimes means one service and sometimes means all services.
- A `scripts/lib/` folder that knows concrete package names, ports, or build output paths for individual services.
