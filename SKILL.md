---
name: bash-service-guide
description: Design, audit, standardize, or refactor Bash-managed service lifecycle scripts / Bash 服务启动停止脚本. Use when Codex needs to create, review, debug, or revise `scripts/setup.sh`, per-service startup or shutdown scripts, `publish` or package installation actions, `start` vs `run` behavior, service and environment selection, port variables, `.run/` logs and PID files, interactive menus, or production start commands that must run repository-installed formal packages instead of development source.
---

# Bash Service Guide

Use this skill to make Bash service scripts consistent, predictable, and easy to operate.

Prefer the repository's existing `AGENTS.md`, `README`, or service docs when they conflict with this guide. Otherwise apply this guide as the baseline.

For any production publish, install, or start path, also apply [Service Release Governance](../service-release-governance/SKILL.md). It owns formal-version, repository, package-installation, and release-gate decisions; this skill owns Bash dispatch and process lifecycle behavior.

Treat "service" as a first-class concept.

Use `scripts/setup.sh` plus per-service scripts when any of these is true:

- the repository has two or more long-running services
- frontend and backend use different startup models or build pipelines
- different services have independent ports, PID files, runtime directories, or production entrypoints
- `setup.sh` already contains large `case "$service"` branches or repeated service-specific conditionals

Read extra references only when needed:

- Read [references/skeleton.md](references/skeleton.md) when you need a starter layout for `scripts/setup.sh` plus per-service scripts.
- Read [references/rules.md](references/rules.md) when you need the full invariant list, production-entrypoint rules, or a review checklist.
- Read [references/runtime-patterns.md](references/runtime-patterns.md) when the repository is Python- or frontend-based and you need concrete `dev` vs `prod` command patterns.
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
5. Align release and runtime behavior.
   - Allow `dev` to run current source directly.
   - Make `publish` build and upload a formal package, and make production installation fetch an exact version from the repository selected by `service-release-governance`.
   - Make `start prod` and `run prod` execute only that installed package. Fail when it is missing; never fall back to source or a local build directory.
6. Verify behavior after editing.
   - Check shell syntax.
   - Smoke-test valid and invalid parsing plus at least one affected command path per touched service.

## Core Invariants

- Expose lifecycle actions through `scripts/setup.sh`.
- Keep `start` backgrounded and `run` foregrounded.
- Split multi-service repositories into a dispatcher plus per-service scripts.
- Resolve `action -> service -> env` in multi-service repositories.
- Require `dev` or `prod` for `start`, `stop`, `restart`, and `run`.
- Store logs, PID files, and lock files under `.run/`, with service-scoped names when needed.
- Define ports at the top of the relevant script and reuse them everywhere.
- Make `start prod` and `run prod` execute the exact formal package installed from the selected repository, never source or local build output.
- Keep `status` non-interactive and make it report every configured environment unless the repository defines an explicit filter.
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
