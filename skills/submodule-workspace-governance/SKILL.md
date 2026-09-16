---
name: submodule-workspace-governance
description: Design, scaffold, or audit a "-dev" orchestration repository that aggregates independently-versioned application repositories as Git submodules under apps/, with root-level scripts/ (init.sh, build.sh, all.sh, setup.sh) that operate across all of them. Use when Codex needs to create a new <product>-dev repo, add or update a submodule under apps/, decide whether a file belongs in the dev repo or an app repo, write or review cross-repo init/build/all scripts, or explain how a multi-repo product (e.g. <product>-api backend + <product>-web frontend + <product>-dev orchestrator, plus optional <product> core-library and <product>-<capability> plugin apps) should be split and wired together.
---

# Submodule Workspace Governance

Use this skill for products split across multiple independently-deployable repositories (typically a backend and a web frontend) that are developed together and need one place to check them out, pin compatible versions, and build them as a set. Each app keeps its own repository, history, issues, and release cadence; the dev repo only composes them.

Read extra references only when needed:

- Read [references/skeleton.md](references/skeleton.md) for a starter layout and template scripts.
- Read [references/rules.md](references/rules.md) for the full invariant list and a review checklist.

## Recognize This Pattern

Use a `<product>-dev` submodule workspace when any of these is true:

- The product already is, or is being split into, two or more independently-deployable repositories (e.g. `<product>-api` backend + `<product>-web` frontend).
- Contributors need to run backend and frontend together from matching, pinned commits during development.
- Releases need a single record of "which commit of each app shipped together" without merging the repositories.

Do not use it for a single-repo product with multiple internal packages or folders — that is a monorepo, and belongs to `project-structure-governance`'s `monorepo` profile (`apps/<name>` as plain directories in one repo, not submodules).

## Repo Roles

| Repo | Owns | Does not own |
| --- | --- | --- |
| `<product>` (core library) | Domain logic and data model shared by every other app; published as an installable package | A running service of its own; cross-repo tooling |
| `<product>-api` (backend/server) | A thin service that depends on `<product>` (and any plugin repos) and exposes it to `<product>-web`; must satisfy the Entrypoint Contract (see [Service Release Governance](../service-release-governance/SKILL.md) / [Bash Service Guide](../bash-service-guide/SKILL.md)) once it has a runnable service | Domain logic that belongs in `<product>` or a plugin repo; other apps' code |
| `<product>-web` (frontend) | Its own source, tests, README, and lifecycle scripts; must satisfy the same Entrypoint Contract for its dev/build/serve commands; must reverse-proxy backend-facing paths (`/api`, health checks, ...) to `<product>-api` | Other apps' code; cross-repo tooling; CORS handling on the `-api` side |
| `<product>-<capability>` (plugin/driver, e.g. `<product>-aliyun`) | A focused integration consumed by `<product>` or `<product>-api`, or published standalone for third parties | A CLI/service entrypoint — it ships as a library, not a process anything starts |
| `<product>-dev` (orchestrator) | Submodule pointers, cross-repo `scripts/`, top-level README describing the set | Any application source of its own |

Only the `-web` and `-api` apps are held to the CLI/service Entrypoint Contract — `<product>` and `<product>-<capability>` repos are installed as dependencies, not started as processes, so forcing a `server start` CLI onto them is spurious. See [references/rules.md](references/rules.md) for the full naming rationale and the CLI-requirement test. `-web` additionally owns reverse-proxying its own runtime traffic to the paired `-api` backend so a browser never calls `-api` cross-origin after the two deploy to different hosts/ports — see [references/rules.md](references/rules.md)'s Web App Proxy Requirement. Register every app repo, CLI-bearing or not, as its own submodule under `apps/`. Never add application code directly to the dev repo outside `apps/` — if code needs to live somewhere new, it needs its own repo and submodule entry, not a folder in the dev repo.

## Follow This Workflow

1. Inspect first.
   - Confirm each app repo already exists (or create it empty, per the org's normal repo-creation process) before wiring it into the dev repo.
   - Check whether the dev repo already has `apps/`, `.gitmodules`, or `scripts/` — do not re-init what already exists.
2. Wire up submodules.
   - From the dev repo root: `git submodule add <url> apps/<app-name>` for each app. This writes `.gitmodules` and stages the submodule automatically — do not hand-edit `.gitmodules` for a new entry.
   - Use the same short name for the submodule path and the app repo (`apps/funlesson-web` for `funlesson-web.git`), so the path alone tells you which repo it tracks.
3. Add cross-repo scripts under `scripts/`.
   - `init.sh` — `git submodule init && git submodule update`. This is the only command a new contributor needs after a plain (non `--recurse-submodules`) clone.
   - `build.sh` — for each app: switch to its tracked branch, then build it (delegate to `funbuild build`/`funbuild install` per [Service Release Governance](../service-release-governance/SKILL.md) when the app's ecosystem supports it); finish with `funbuild push` at the dev-repo level so the updated submodule pointers get committed and pushed together. Once `setup.sh` exists, `build.sh` can simply be `exec scripts/setup.sh build all` instead of duplicating the loop.
   - `all.sh` (optional) — a plain batch loop for non-service git operations across every app (`status`, `pull`, `test`), not an action/target dispatcher. Add it only once the workspace actually needs one command fanned out across every app in a defined order. Do not add it speculatively.
   - `setup.sh` (optional, add once the workspace has at least one CLI-bearing app) — the cross-repo `<action> <target>` dispatcher for service and build actions. See "Design scripts/setup.sh" below.
   - See [references/skeleton.md](references/skeleton.md) for concrete script bodies.
4. Write the dev repo README.
   - A table of `apps/<name>` -> repo link -> one-line purpose.
   - Clone instructions covering both `git clone --recurse-submodules <dev-repo-url>` and the plain-clone-then-`scripts/init.sh` path.
   - The submodule update sequence from the next section.
   - The `scripts/build.sh` invocation and what it does.
5. Give each new app repo a real README, not a placeholder.
   - State the app's purpose and its relationship to sibling apps (e.g. "Web interface for `<product>-api`, see `<product>-dev` for the paired backend").
   - Keep implementation-detail claims (ports, endpoints, commands) out until the app actually implements them; say the app is in early development instead of inventing behavior.

## Design scripts/setup.sh

Once the workspace has at least one CLI-bearing app (`-web` or `-api`), give `scripts/setup.sh` the same `<action> <target>` shape `bash-service-guide` uses inside each app, one level up:

```
scripts/setup.sh <action> <target>
```

- `<target>` is a short app alias (`api`, `web`, ...) or `all`. Map aliases to `apps/<product>-api` / `apps/<product>-web` in one place in the script so the rest of the dispatch logic never hardcodes a path.
- `<action>` splits into two groups with different `all` semantics — spell this out in the script's usage text, since it is the single most common source of confusion in this pattern:
  - **Service actions** (`start`, `stop`, `restart`, `run`, `status`, `install`, `publish`): only apply to CLI-bearing apps. `all` means "every CLI-bearing app" (`api` + `web`), not every submodule. Delegate each one straight to that app's own `scripts/setup.sh <action>` — do not reimplement PID/port handling at the dev-repo level; the app's own script already owns that per [Bash Service Guide](../bash-service-guide/SKILL.md).
  - **`build`**: applies to every app under `apps/`, CLI-bearing or not — a core-library or plugin app still needs `funbuild build`/`funbuild install` even though nothing ever `start`s it. `all` means every submodule in `.gitmodules`. Finish with one `funbuild push` at the dev-repo level regardless of how many apps were built, so the pointer bump lands as a single commit.
- Reject an unrecognized `<target>` for a service action with a clear error instead of silently no-op'ing — running `start` against a core-library or plugin app is a usage mistake, not something to skip quietly.
- See [references/skeleton.md](references/skeleton.md) for a concrete `setup.sh` body.

## Update Submodules Correctly

Submodule pointers and app-repo commits are two different commits in two different repos — never let one move without the other.

1. Make the change inside the app repo (`apps/<name>`), commit, and push it to that app's own remote first.
2. From the dev repo root, `git add apps/<name>` to stage the moved pointer (repeat per changed app), then commit and push the dev repo.
3. Never leave an app submodule with local commits that were pushed to its own remote but whose pointer update was never committed in the dev repo — that silently desyncs "what the dev repo says shipped together" from reality.
4. Never edit files inside `apps/<name>` and commit only at the dev-repo level — a submodule's working tree changes must be committed in the app repo itself; the dev repo can only record which app commit it points at.
5. Pull latest for all apps with `git submodule update --remote`, then repeat step 2 to record the result.

## Core Invariants

- One app repo per Git submodule under `apps/<app-name>`; no application source lives directly in the dev repo.
- `.gitmodules` entries are only ever produced by `git submodule add`/`git submodule sync`, never hand-written from scratch.
- `scripts/init.sh` is the minimum viable cross-repo script: init + update, nothing else required.
- `all.sh` and `setup.sh` stay optional; add them only when the workspace genuinely needs batch operations or a single entrypoint, per the same rule `bash-service-guide` applies to service scripts.
- Where `setup.sh` exists, `all` means different things for different actions: for service actions (`start`/`stop`/`restart`/`run`/`status`/`install`/`publish`) it scopes to CLI-bearing apps only (`api` + `web`); for `build` it scopes to every app under `apps/`, including core-library and plugin apps. Never conflate the two.
- A submodule pointer change and the corresponding app-repo commit are always committed together as two commits in two repos, app repo first.
- Each app repo keeps its own lifecycle scripts and release process — this skill governs composition, not what happens inside an app. Defer single-repo internal layout to `project-structure-governance` and single-service start/stop/install/publish behavior to `bash-service-guide` and `service-release-governance`.
- The CLI/service Entrypoint Contract applies to the `-web` and `-api` apps only; a core library repo (`<product>`) or a plugin/driver repo (`<product>-<capability>`) stays exempt because nothing starts it as a process.
- `<product>-web`'s own runtime server always reverse-proxies backend-facing paths to `<product>-api` — it is never just a static-file server once the two deploy separately. See [references/rules.md](references/rules.md)'s Web App Proxy Requirement.

## Validate Before Finishing

- `git submodule status` from the dev repo root shows no unexpected `+` (pointer ahead of what's committed) or `-` (submodule not initialized) markers for apps you touched.
- Every `apps/<name>` entry in `.gitmodules` has a matching row in the dev repo README's app table.
- `bash -n` any touched shell script.
- If you changed an app repo, confirm its own commit is pushed before you commit the dev repo's pointer update.
- If you touched `<product>-web`, confirm it still proxies backend-facing paths to `<product>-api` (not just serving static assets) and that dev-time tooling proxies the same backend URL as the production server.
