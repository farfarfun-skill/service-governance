# Decision Tree

Read this file when the task is not just "write a script", but "decide how to change or review one".

## Paths

### Create

Use this path when:
- `scripts/setup.sh` does not exist
- The existing script is a stub
- The repository has lifecycle commands but no unified entrypoint

Then:

1. Read [skeleton.md](skeleton.md) and [rules.md](rules.md)
2. Read [runtime-patterns.md](runtime-patterns.md) if runtime commands matter
3. Implement the smallest command set the repository actually supports
4. If the repo has multiple long-running services, create a dispatcher plus per-service scripts
5. Add `all` only if the repository actually needs batch operations

### Refactor

Use this path when:
- `scripts/setup.sh` exists but mixes `dev` and `prod` incorrectly
- `scripts/setup.sh` assumes a single service but the repository now has frontend and backend apps, or several service targets
- `start` and `run` semantics are blurred
- Ports, logs, or PID handling are scattered
- Prod uses a dev-only command

Then:

1. Preserve supported commands unless docs say otherwise
2. Normalize parsing and dispatch first
3. Split into per-service scripts when one file is managing several targets
4. Centralize ports and runtime-file paths near the top of the relevant script
5. If `all` exists, make its order and failure policy explicit

### Review

Use this path when the user asks for analysis, audit, or optimization without explicitly asking for a rewrite.

Prioritize findings in this order:

1. Incorrect prod runtime behavior
2. Broken lifecycle semantics such as background `run` or non-detached `start`
3. Unsafe or inconsistent PID / log / port handling, especially stale PIDs, duplicate starts, or cross-service collisions
4. Ambiguous CLI parsing or menu flow
5. Unsafe command construction, signal loss, or wrapper PIDs
6. Structure issues, such as a multi-service repo still forced through one oversized script
7. Aggregate `all` behavior that is implicit, inconsistent, or operationally unsafe

Report findings with concrete references when possible: wrong command path, affected service or environment, and likely operational failure.

## CLI Parsing Contract

- `scripts/setup.sh start prod`: in a single-service repo, run directly without prompts
- `scripts/setup.sh start api prod`: in a multi-service repo, run directly without prompts
- `scripts/setup.sh start all dev`: only if the repository explicitly supports aggregate operations
- `scripts/setup.sh start`: resolve missing information interactively
- `scripts/setup.sh`: resolve action first, then service if required, then environment if required
- `scripts/setup.sh status api`: report every configured environment for `api` without prompting

Reject or handle clearly: unknown actions, unknown services, extra positional arguments, invalid environments, and unsupported use of `all`.

Do not open Gum menus when CLI args already specify a valid action, service, and environment.

## Dispatch Contract

- Parse args first
- Validate action
- Validate or resolve service before environment when the command targets one service
- If `all` is supported, resolve it as a separate dispatch path rather than pretending it is a normal service script
- Validate or resolve environment only for environment-bound actions
- Reject extra args and invalid supplied values; never turn them into interactive prompts
- Call `do_<action>` directly in single-service scripts, or call a per-service script from `setup.sh` in multi-service layouts

Example mental model:

```text
action -> needs service? -> resolve service -> needs env? -> resolve env -> dispatch -> verify runtime files and command choice
```

## Review Questions

- Does CLI usage work non-interactively for automation?
- Does interactive mode only fill in missing information?
- In multi-service repos, are `status` and `stop` consistent with the PID and port created by `start` for the selected service?
- Are PID liveness, stale state, startup failure, and graceful shutdown handled explicitly?
- Does foreground execution preserve signals and exit status without string evaluation?
- If `all` exists, is its service order and error handling obvious from reading the script?
- Is the script set easier to reason about after the change, or just spread across more files without cleaner boundaries?
