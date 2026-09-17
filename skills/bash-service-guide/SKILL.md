---
name: bash-service-guide
description: Design, audit, standardize, or refactor Bash-managed service lifecycle scripts / Bash 服务启动停止脚本. Use when Codex needs to create, review, debug, or revise `scripts/setup.sh`, per-service startup or shutdown scripts, `publish` or package installation actions, `start` vs `run` behavior, service and environment selection, port variables, `.run/` logs and PID files, interactive menus, or production start commands that must run repository-installed formal packages instead of development source.
---

# Bash Service Guide

Use this skill to make Bash service scripts consistent, predictable, and easy to operate.

Prefer the repository's existing `AGENTS.md`, `README`, or service docs when they conflict with this guide. Otherwise apply this guide as the baseline.

For any production publish, install, or start path in an npm or Python service, also apply [Service Release Governance](../service-release-governance/SKILL.md). It owns formal-version, repository, package-installation, and release-gate decisions; this skill owns Bash dispatch and process lifecycle behavior. That skill's concrete rules assume an npm/PyPI-style registry install and don't cover other ecosystems (e.g. the Flutter Web pattern in [references/runtime-patterns.md](references/runtime-patterns.md), which publishes through `funbuild`/`funpub` to a private generic artifact store instead of a language registry) — reproduce the same underlying principles (immutable versioned artifacts, no dev/prod source mixing, no serving straight from the checkout) through that ecosystem's actual tooling instead of assuming service-release-governance's registry-specific commands apply.

Treat "service" as a first-class concept.

Use `scripts/setup.sh` plus per-service scripts when any of these is true:

- the repository has two or more long-running services
- frontend and backend use different startup models or build pipelines
- different services have independent ports, PID files, runtime directories, or production entrypoints
- `setup.sh` already contains large `case "$service"` branches or repeated service-specific conditionals

Read extra references only when needed:

- Read [references/skeleton.md](references/skeleton.md) when you need a starter layout for `scripts/setup.sh` plus per-service scripts.
- Read [references/rules.md](references/rules.md) when you need the full invariant list, production-entrypoint rules, or a review checklist.
- Read [references/runtime-patterns.md](references/runtime-patterns.md) when the repository is Python-, frontend-, or Flutter-web-based and you need concrete `dev` vs `prod` command patterns.
- Read [references/decision-tree.md](references/decision-tree.md) when you need to choose between create, refactor, or review paths, or when CLI parsing behavior is ambiguous.

## Follow This Workflow

1. Inspect the repository first.
   - Find existing `scripts/setup.sh`, helper scripts, service entrypoints, and docs.
   - Identify whether the repository is single-service or multi-service.
   - Identify the runtime type for each service.
   - Inventory the actions, production artifacts, and process-management conventions each service actually supports.
2. Normalize the public interface.
   - Expose user-facing lifecycle actions through `scripts/setup.sh`.
   - In multi-service repositories, keep `setup.sh` as a dispatcher and move service-specific process logic into separate scripts.
   - Treat `all` as optional. Add it only when the repository actually needs batch operations.
   - Hide unsupported commands instead of leaving stubs.
3. Keep parsing deterministic.
   - Resolve `action -> service -> env` in multi-service repositories.
   - Resolve `action -> env` in single-service repositories.
   - Use CLI args when provided; prompt only for missing pieces and reject invalid or extra args.
4. Make runtime state deterministic.
   - Keep logs, PID files, and lock files under `.run/`.
   - Keep ports in a top configuration block.
   - Refuse duplicate starts, distinguish stale PID files from live processes, and stop only the recorded service process.
5. Align release and runtime behavior with `service-release-governance`'s unified dev/prod contract — both modes install a package and start it through the package's own CLI; only the package source and `--config` path differ, and only `install-dev`, `install-prod`, and `publish` are environment-asymmetric by name.
   - Make `install-dev` (dev-only, no argument) clear any previous local build/install, rebuild from the current working tree, and install the built package locally (`funbuild install` when available) — it always means the local-build flavor.
   - Make `install-prod [version]` (prod-only) pull the formal package from the registry/index directly via the ecosystem's package manager (`pip install <pkg>[==<version>]`, `npm install -g <pkg>[@<version>]`) — the bootstrap path for a host that has nothing installed yet, since the CLI cannot install itself before it exists. Without a version it is idempotent: a no-op if any version is already installed, never silently upgrading. With a version it pins to exactly that version.
   - Make `publish` (prod-only, no argument) build and upload a formal package. Call `funbuild build` (alias `funbuild release`) when the repository is one it auto-detects (npm/pnpm/yarn, uv, Poetry, or hybrid) instead of hand-rolling a build/publish sequence.
   - Make `start`, `stop`, `restart`, `run`, and `status` take no `dev`/`prod` argument at all — they operate on whichever package is currently installed on the host, and they are thin delegations to the installed CLI's own `server` subcommands rather than something the Bash script re-implements. `start` calls `<cli> server start [--config <path>] [--port ...]` — the CLI itself detaches/backgrounds and writes its own PID file next to its resolved config path, so the script does not `nohup` it or manage a separate PID file. `run` calls `exec <cli> server run [--config <path>] [--port ...]` — the CLI's foreground counterpart, so signals and exit status propagate. `stop` and `status` call `<cli> server stop`/`<cli> server status` directly; `status` reads that same CLI-owned PID file, while `stop`'s actual termination step should go through `funshell port <port> --kill` internally (per `service-release-governance`'s Entrypoint Contract) rather than hand-signaling the PID from that file. Fail when the required install is missing; never fall back to source or a local build directory silently. Make `status` also report the installed package's version, not just PID/port.
   - Treat `upgrade [version]` and `rollback <version>` as calling the installed CLI's own top-level subcommands (`<cli> upgrade [version]`, `<cli> rollback <version>`) when it implements them, falling back to the ecosystem's package manager directly (`pip install <pkg> -U` / `pip install <pkg>==<version>`) only when it doesn't. Both assume a package is already installed — neither is a bootstrap path; `install-prod` is. Treat `uninstall` the same way (`<cli> uninstall`); add thin passthroughs in the lifecycle script only when it genuinely needs one consistent entrypoint. `uninstall` must stop the running service first (`do_stop`, i.e. `<cli> server stop`), then remove the package — never remove a package out from under a live process.
6. Verify behavior after editing.
   - Check shell syntax.
   - Smoke-test valid and invalid parsing plus at least one affected command path per touched service.

## Core Invariants

- Expose lifecycle actions through `scripts/setup.sh`.
- Keep `start` backgrounded and `run` foregrounded.
- Split multi-service repositories into a dispatcher plus per-service scripts.
- Resolve `action -> service -> env` in multi-service repositories.
- The `install-dev`/`install-prod` split and `publish` (always prod) already encode environment in the action name, so none of them take a separate `dev`/`prod` argument (`install-prod` optionally takes a version instead). `start`, `stop`, `restart`, `run`, and `status` take no `dev`/`prod` argument either.
- Let the installed CLI own PID files: `start`/`run` call `<cli> server start`/`<cli> server run` directly, and the CLI backgrounds itself (for `start`) and writes its own `<cli-name>.pid` next to its resolved config path — the Bash script does not `nohup` it or keep a separate `.run/<service>.pid` for services built to this contract. `stop`/`status` call `<cli> server stop`/`<cli> server status`; `status` reads that same file, while `stop` should terminate via `funshell port <port> --kill` internally rather than hand-signaling the PID from it. Keep the older Bash-owned `.run/` PID-file pattern, with `funshell port <port> --kill` (falling back to PID + `TERM`) as its stop mechanism, only as a documented fallback for a CLI that cannot daemonize itself.
- Define ports at the top of the relevant script and reuse them everywhere.
- Make `start` and `run` execute the installed package's CLI (`<cli> server start [--config <path>]` / `<cli> server run [--config <path>]`) against whichever package is currently installed — a locally built-and-installed copy on a dev host, the exact formal package on a prod host — never source or local build output run directly.
- Keep `status` non-interactive and make it report the installed package's version alongside PID/port.
- Make `uninstall`, when the script owns it, stop the running service first and only then remove the package.
- Prefer CLI args over prompts; use the repository's existing prompt tool only as a fallback when args are absent.

Use [references/rules.md](references/rules.md) when you need the expanded version of these rules.

## Structure Guidance

Prefer one of these layouts:

1. Single service
   - `scripts/setup.sh`
2. Multiple services
   - `scripts/setup.sh`
   - `scripts/services/<service>.sh`
   - optional shared helpers under `scripts/lib/`

For multi-service repositories, keep `setup.sh` focused on argument parsing, target selection, and dispatch. Keep ports, runtime files, and concrete start/stop logic inside each service script.

Use `scripts/lib/` only for shared mechanics such as logging, PID checks, and port probes.

Read [references/skeleton.md](references/skeleton.md) for a starter skeleton. Match paths, binary names, and package entrypoints to the target repository instead of copying placeholders literally.

## Apply These Implementation Rules

- Resolve repository and service roots relative to the script location, not the current working directory.
- Keep a clear mapping from logical service name to service script and service root.
- Keep aggregate operations such as `all` explicit and rare; define order and failure behavior instead of improvising.
- Quote variable expansions unless unquoted form is explicitly required.
- Prefer explicit helper functions when they simplify branching.
- Write PID files only for backgrounded processes that the script is responsible for managing.
- Use Bash arrays for commands, use `exec` for foreground execution, and avoid `eval` or shell command strings.
- Treat a port probe as supporting evidence, not proof that a PID belongs to the managed service.
- When replacing an existing script, preserve any repository-specific safeguards that are still valid.

## Validate Before Finishing

- Run `bash -n scripts/setup.sh` after shell edits.
- If helper scripts were touched, syntax-check them too.
- In multi-service layouts, syntax-check each touched service script.
- If the repository provides a smoke-test command, use it.
- When practical, test both a foreground path (`run`) and a background path (`start`/`stop`) for each modified service.

Use [references/rules.md](references/rules.md) as a final review checklist when the script set is large or heavily refactored.
