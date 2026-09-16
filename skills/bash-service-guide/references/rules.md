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

`start`/`run`/`stop`/`status` are thin delegations to the installed CLI's own `server` subcommands, not something the Bash script re-implements:

- `start`: call `<cli> server start [--config <path>]` directly. The CLI itself backgrounds/detaches and writes its own `<cli-name>.pid` next to its resolved config path — the script does not `nohup` this call or keep a separate PID file for services built to this contract.
- `run`: call `exec <cli> server run [--config <path>]` — the CLI's foreground counterpart of `start` (identical startup and PID file, but stays attached to the terminal instead of detaching); `exec` so signals and exit status propagate.
- `stop`: call `<cli> server stop`. The CLI reads its own PID file (next to its resolved config path) and stops that process.
- `restart`: implement as `stop` + `start` (i.e. `do_stop` then `do_start`).
- `status`: call `<cli> server status`. Report the service's state without prompting, including the installed package's version (e.g. via `pip show`/`npm ls -g`/the CLI's own `--version`) alongside PID/port.
- `publish`: build and upload a new formal package; prod-only, always the release flavor. Call `funbuild build` (alias `funbuild release`) when the project uses this convention — it already covers version bump, build, publish, push, and tag for npm, Python, and hybrid repos.
- `install`: dev-only, always the local-build flavor — clear the previous local build/install, rebuild from the current working tree, and install the built package locally (`funbuild install` when available). There is no `prod` counterpart in the lifecycle script: a prod host installs the exact formal version directly via the ecosystem's package manager or the installed CLI's own `upgrade`/`rollback` subcommand, per `service-release-governance`, not through this script.
- `upgrade`/`rollback <version>`/`uninstall`: primarily the installed CLI's own top-level subcommands (`<cli> upgrade`, `<cli> rollback <version>`, `<cli> uninstall`), not something to hand-roll in Bash. Add a passthrough in the lifecycle script only when it genuinely needs one consistent entrypoint for all of these. `uninstall` must stop the running service first, then remove the package — never remove it out from under a live process. There is no separate `install`-for-rollback path; rolling back means `rollback <version>`.
- `all`: optional aggregate target, not a required baseline feature.

A CLI that cannot daemonize itself is the exception, not the baseline: only then does the Bash script fall back to `nohup`-backgrounding `server start` in the foreground and managing its own `.run/<service>.pid` file, per the fallback pattern in [runtime-patterns.md](runtime-patterns.md#backgrounding-pattern).

Keep `publish`, `install`, and any `upgrade`/`rollback`/`uninstall` passthrough separate from `start`/`run`; a start must not silently build, publish, install, or change the installed version. Expose these only when the Bash lifecycle script actually owns that operation. Otherwise leave them to the installed CLI and release automation, and treat the installed package as a prerequisite for `start`/`run`.

Do not expose commands that the service does not actually support. Apply [Service Release Governance](../../service-release-governance/SKILL.md) to every production publish, install, and start path.

Use `exec` for `run`'s final foreground command (`exec <cli> server run ...`) so signals and exit codes reach the service directly. For `start`, the installed CLI is expected to self-daemonize and manage its own PID file (see [Runtime Files](#runtime-files)); only fall back to the repository's own backgrounding/PID mechanism when the CLI genuinely cannot daemonize itself.

### Selection Order

- Treat the target service as explicit state when the repository contains multiple long-running services.
- Preserve the parse order `action -> service`. No baseline action takes an environment argument, so there is no further `-> env` step to resolve.
- Single-service repositories may omit the service argument.
- If aggregate mode is supported, treat `all` as an explicit pseudo-service rather than an implicit default.

### Runtime Files

- The installed CLI owns its own PID file: `<cli> server start`/`<cli> server run` write `<cli-name>.pid` next to whichever config path they resolved (the CLI's hardcoded default, or an explicit `--config` override). The Bash script does not create or manage this file for services built to this contract — it only needs to know the convention well enough to document or inspect it.
- Still create `.run/` under the service root for anything the Bash script itself remains responsible for — e.g. a repo-tracked dev config file, or, under the non-daemonizing-CLI fallback, the script's own PID and log files.
- If the fallback pattern is in use and one dispatcher manages several services, either:
  - write fallback runtime files into each service root's `.run/`, or
  - use a repo-level `.run/` with service-qualified filenames such as `api.pid`
- Keep any fallback filenames specific enough to avoid collisions across services. No environment suffix is needed — a host only ever has one active install per service.
- Avoid `/tmp` unless the repository explicitly requires it.

### Process Ownership

Once the installed CLI backgrounds itself and owns its own PID file (see [Runtime Files](#runtime-files)), the CLI is responsible for upholding the following, and the Bash script only propagates its exit code:

- Before `start`/`run`, reject a live PID file and remove or report a stale one explicitly.
- After backgrounding, write the captured PID atomically and verify that it survives a short startup grace period.
- Redirect stdin, stdout, and stderr for background starts so the process is detached from the terminal.
- Before `stop`, require a numeric PID, check that it is live, and verify service identity when a reliable command or metadata check is available.
- Send `TERM`, wait for a bounded interval, and remove the PID file only after the process exits. Use `KILL` only when policy explicitly permits forced shutdown.
- Never discover a process by port and then kill it. A listener can belong to an unrelated process.

Under the non-daemonizing-CLI fallback, the Bash script must uphold all of the above itself, plus serialize lifecycle changes with a service-scoped lock when concurrent invocations are realistic.

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

- `start` uses `<installed-cli> server start [--config <path>] [--port ...]`; `run` uses the CLI's foreground counterpart, `exec <installed-cli> server run [--config <path>] [--port ...]`. Both write the CLI's own `<cli-name>.pid` next to whichever config path got resolved, so the script does not manage a separate PID file. Whether the underlying package came from a local build or the registry-pinned version is decided entirely at install time, not by an argument to `start`/`run`.
- `--config` is optional. When passed, it typically points `install`-time dev usage at a repo-tracked file such as `config/dev.toml`; when omitted, the CLI resolves its own hardcoded default (`${XDG_CONFIG_HOME:-~/.config}/<org>/<cli-name>/config.toml`), which is normally sufficient for a prod host. The PID file follows whichever path is actually in effect.
- Ad hoc hot-reload tooling (`npm run dev`, `vite dev`, `next dev`, `uv run --reload`) is fine as an unmanaged workflow while actively editing code, but is never what the lifecycle script's `start`/`run` itself invokes — that action must go through the CLI, backed by a package actually installed from the current source.
- For Python services, run the installed CLI entrypoint (`<cli> server start`), never a raw module path, script, editable install, `PYTHONPATH` override, or an implicit `uv run` workspace resolution.
- For frontend services, run the installed CLI/production server from the exact installed package, not the checkout's `dist/`/`build/` directory run directly.
- In mixed frontend/backend repositories, choose the install and start commands independently for each service.
- Fail `start`/`run` when the required installed package is absent; never fall back to source or a stale install.

## Review Checklist

- Does `setup.sh` own the user-facing lifecycle interface?
- If the repo has multiple services, is service selection explicit and deterministic?
- If the repo has multiple services, is the split threshold met, or is one large script still carrying too much service-specific logic?
- Does `start` call `<cli> server start` and let the CLI detach and own its PID file, rather than the script re-implementing `nohup`/PID handling (except under the documented non-daemonizing-CLI fallback)?
- Does `run` call `<cli> server run` — not `server start` — and stay in the foreground via `exec`, preserving service signals and exit status?
- Do `start`, `stop`, `restart`, `run`, and `status` correctly take no `dev`/`prod` argument, leaving environment identity entirely to what's installed?
- Is the menu flow action-first and service-second when needed?
- If `all` exists, are service order, output format, and failure policy explicit?
- Are ports centralized at the top of the relevant script?
- Are status and health checks derived from the same service-specific port configuration?
- Are duplicate starts, stale PID files, failed starts, and graceful stop timeouts handled explicitly (by the CLI itself, or by the script only under the fallback pattern)?
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
- Re-implementing `nohup`/PID-file management in Bash for a CLI that already daemonizes itself and owns its own PID file.
- Making Bash `run` call `<cli> server start` instead of the CLI's dedicated foreground subcommand, `<cli> server run`.
- Publishing a wrapper CLI package (e.g. the embedded npm CLI in [runtime-patterns.md](runtime-patterns.md#providing-the-missing-cli-a-self-published-npm-wrapper)) with `"private": true` still set from scaffolding defaults, which blocks `npm publish` unconditionally.
- Trusting bare PID existence as proof a backgrounded process is still the one that was started, instead of cross-checking its command line — PIDs get reused by unrelated processes.
- Letting a wrapper CLI's own version (e.g. the embedded npm CLI's semver) drift independently of the artifact it serves with no pinned pairing recorded anywhere — "install the matching CLI version" becomes a manual guess instead of a lookup.

## Anti-Patterns

- A single `setup.sh` that handles `api`, `web`, `admin`, and `worker` entirely through nested `case` blocks.
- Repeated `cd` hopping across several directories to infer which service is being operated.
- One shared PID file or one shared log file for several services.
- Hiding aggregate behavior behind normal commands so `start` sometimes means one service and sometimes means all services.
- A `scripts/lib/` folder that knows concrete package names, ports, or build output paths for individual services.
- A sibling `<app>-cli` package, versioned and published independently of the app it deploys, instead of embedding the CLI inside that app's own directory — this turns "which CLI version matches which build" into something that has to be tracked separately.
