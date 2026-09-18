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

This table shows the conventional roles for a two/four-app product; the underlying classification is by category, not by name — see [references/rules.md](references/rules.md)'s App Category & Lifecycle Requirement for the general rule and how to classify an app that doesn't fit this table (e.g. a worker service not named `-api`/`-web`, or a `-web`/`-api` app that turns out to have no process of its own).

Service apps — conventionally `-web`/`-api`, but any app that something starts as a long-running process — are held to the full CLI/service Entrypoint Contract (service actions plus release actions). Package apps — conventionally `<product>` and `<product>-<capability>` — get release actions only, since they're installed as dependencies, not started as processes; forcing a `server start` CLI onto one is spurious. Build and publish are not part of this dev repo's own action dispatch for any app category — `funbuild build`, run from the dev repo root, owns building and publishing every app under `apps/` on its own (see `scripts/funbuild.toml` below). Nested-workspace apps (an `apps/<name>` that is itself a `<something>-dev`) are excluded from this dev repo's own action dispatch entirely — they own and dispatch their own classification recursively. See [references/rules.md](references/rules.md) for the full naming rationale, the per-app classification test, and why the category must be declared explicitly (in code and in the README) rather than inferred from the repo's name suffix. `-web` additionally owns reverse-proxying its own runtime traffic to the paired `-api` backend so a browser never calls `-api` cross-origin after the two deploy to different hosts/ports — see [references/rules.md](references/rules.md)'s Web App Proxy Requirement. Register every app repo, whatever its category, as its own submodule under `apps/`. Never add application code directly to the dev repo outside `apps/` — if code needs to live somewhere new, it needs its own repo and submodule entry, not a folder in the dev repo.

## Follow This Workflow

1. Inspect first.
   - Confirm each app repo already exists (or create it empty, per the org's normal repo-creation process) before wiring it into the dev repo.
   - Check whether the dev repo already has `apps/`, `.gitmodules`, or `scripts/` — do not re-init what already exists.
2. Wire up submodules.
   - From the dev repo root: `git submodule add <url> apps/<app-name>` for each app. This writes `.gitmodules` and stages the submodule automatically — do not hand-edit `.gitmodules` for a new entry.
   - Use the same short name for the submodule path and the app repo (`apps/funlesson-web` for `funlesson-web.git`), so the path alone tells you which repo it tracks.
3. Add cross-repo scripts under `scripts/`.
   - `init.sh` — `git submodule init && git submodule update`. This is the only command a new contributor needs after a plain (non `--recurse-submodules`) clone.
   - `build.sh` — `funbuild build`, run from the dev repo root, already recognizes this layout (an `apps/` directory plus `scripts/funbuild.toml`) and fans out on its own: bump the shared version, build/publish every non-nested-workspace app under `apps/`, then commit/push/tag the dev repo once. `build.sh` therefore stays a one-line wrapper: `exec funbuild build`.
   - `all.sh` (optional) — a plain batch loop for non-service git operations across every app (`status`, `pull`, `test`), not an action/target dispatcher. Add it only once the workspace actually needs one command fanned out across every app in a defined order. Do not add it speculatively.
   - `setup.sh` (required, alongside `init.sh`/`build.sh`) — the cross-repo `<action> <target>` dispatcher, covering service lifecycle (`start`/`stop`/`restart`/`run`/`status`) for service apps and release lifecycle (`install-dev`/`install-prod`/`upgrade`/`rollback`) for service + package apps (nested-workspace apps excluded from both). Build and publish are not part of this dispatcher — `funbuild build` handles both on its own. Every `<product>-dev` repo ships this from the start, same as `init.sh`/`build.sh` — it is not something to add later once a service app shows up. See "Design scripts/setup.sh" below.
   - `scripts/funbuild.toml` (required) — declares the version shared by every app under `apps/`; `funbuild build` reads and bumps it so all apps publish under one version number.
   - See [references/skeleton.md](references/skeleton.md) for concrete script bodies.
4. Write the dev repo README.
   - A table of `apps/<name>` -> repo link -> one-line purpose.
   - Clone instructions covering both `git clone --recurse-submodules <dev-repo-url>` and the plain-clone-then-`scripts/init.sh` path.
   - The submodule update sequence from the next section.
   - The `scripts/build.sh` invocation and what it does.
5. Give each new app repo a real README, not a placeholder.
   - State the app's purpose and its relationship to sibling apps (e.g. "Web interface for `<product>-api`, see `<product>-dev` for the paired backend").
   - Keep implementation-detail claims (ports, endpoints, commands) out until the app actually implements them; say the app is in early development instead of inventing behavior.
6. Create `docs/` in the dev repo, following `project-structure-governance`'s document-bundle-standard layout (`product/`, `design/`, `development/`, `testing/`, `retrospective/`, `release/`, each `<feature-slug>` directory starting from `001-overview.md`). See [references/skeleton.md](references/skeleton.md)'s `docs/` section. Individual `apps/<name>` repos do not get their own `docs/` — cross-app and product-level documentation centralizes here, except for an app that is itself a nested `<something>-dev` workspace, or a dedicated `apps/<product>-doc` repo.

## Design scripts/setup.sh

Every `<product>-dev` repo gets a `scripts/setup.sh` from the start — give it the same `<action> <target>` shape `bash-service-guide` uses inside each app, one level up:

```
scripts/setup.sh <action> <target>
```

- `<target>` is a short app alias (`api`, `web`, ...) or `all`. Map aliases to `apps/<name>` in one place in the script so the rest of the dispatch logic never hardcodes a path — and never infer an alias's category from its text; register it explicitly (see [references/rules.md](references/rules.md)'s App Category & Lifecycle Requirement).
- `<action>` splits into two groups with different `all` semantics — spell this out in the script's usage text, since it is the single most common source of confusion in this pattern:
  - **Service actions** (`start`, `stop`, `restart`, `run`, `status`): only apply to service apps. `all` means "every service app", not every submodule. Delegate each one straight to that app's own `scripts/setup.sh <action>` — do not reimplement PID/port handling at the dev-repo level; the app's own script already owns that per [Bash Service Guide](../bash-service-guide/SKILL.md).
  - **Release actions** (`install-dev`, `install-prod`, `upgrade`, `rollback`): apply to every service app and every package app — a core-library or plugin app still needs these even though it has no service actions.
  - `build` and `publish` are not `setup.sh` actions at all — `funbuild build`, run from the dev repo root, owns both across every app under `apps/` on its own (see the `scripts/funbuild.toml` bullet in "Follow This Workflow" above).
  - Nested-workspace apps are excluded from both groups — bump their pointer like any other submodule, but dispatch into them (if needed) by `cd`-ing into the nested workspace and running its own `scripts/setup.sh`, not through this script's aliases.
- Reject an unrecognized `<target>`, or a target outside an action's group, with a clear error instead of silently no-op'ing — running `start` against a package or nested-workspace app is a usage mistake, not something to skip quietly.
- A missing `<action>` or `<target>` should fall back to an interactive `gum choose` menu rather than erroring outright — reuse `bash-service-guide`'s `choose()` helper instead of a second interactive style. This is the one cross-repo script that needs `#!/usr/bin/env bash` (arrays, `[[ ]]`) instead of POSIX `sh`; every fully-specified invocation still skips `gum` entirely.
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
- `scripts/init.sh`, `scripts/build.sh`, `scripts/setup.sh`, and `scripts/funbuild.toml` are all required in every dev repo — `init.sh` is init + update, nothing else; `build.sh` is `exec funbuild build`, which fans out build/publish to every app under `apps/` on its own; `scripts/funbuild.toml` declares the version that fan-out shares across every app; `setup.sh` is the cross-repo entrypoint for service lifecycle (`start`/`stop`/`restart`/`run`/`status`) and release lifecycle (`install-dev`/`install-prod`/`upgrade`/`rollback`) — build and publish are `funbuild`'s job, not `setup.sh`'s — so nobody has to `cd` into an app or hand-roll `funbuild`/CLI invocations from the dev repo root. `all.sh` stays optional — add it only when the workspace genuinely needs a plain batch loop beyond what `setup.sh` already dispatches.
- In `setup.sh`, `all` scopes by app category, not by action name alone: service actions (`start`/`stop`/`restart`/`run`/`status`) scope to service apps only; release actions (`install-dev`/`install-prod`/`upgrade`/`rollback`) scope to service + package apps; nested-workspace apps are excluded from both action groups. Never conflate the two.
- A submodule pointer change and the corresponding app-repo commit are always committed together as two commits in two repos, app repo first.
- Each app repo keeps its own lifecycle scripts and release process — this skill governs composition, not what happens inside an app. Defer single-repo internal layout to `project-structure-governance` and single-service start/stop/install/publish behavior to `bash-service-guide` and `service-release-governance`.
- The CLI/service Entrypoint Contract's service actions apply to service apps only (conventionally `-api`/`-web`, but any app something starts as a long-running process); package apps (conventionally `<product>` and `<product>-<capability>`) still get release actions since they're installed artifacts, just not started as processes; nested-workspace apps are excluded entirely and self-govern. Classify by the test in [references/rules.md](references/rules.md), never by name suffix alone, and keep the code (`resolve_service_app`/`resolve_release_app`) and README app table in agreement.
- `<product>-web`'s own runtime server always reverse-proxies backend-facing paths to `<product>-api` — it is never just a static-file server once the two deploy separately. See [references/rules.md](references/rules.md)'s Web App Proxy Requirement.
- `docs/` is required in the dev repo and follows `project-structure-governance`'s document-bundle-standard layout. App repos under `apps/` do not maintain their own `docs/` — that content centralizes in the dev repo — except an app that is itself a nested `<something>-dev` workspace, or a dedicated `apps/<product>-doc` repo.

## Validate Before Finishing

- `git submodule status` from the dev repo root shows no unexpected `+` (pointer ahead of what's committed) or `-` (submodule not initialized) markers for apps you touched.
- Every `apps/<name>` entry in `.gitmodules` has a matching row in the dev repo README's app table.
- `scripts/init.sh`, `scripts/build.sh`, `scripts/setup.sh`, and `scripts/funbuild.toml` all exist; `scripts/setup.sh --help`/usage output covers `start`/`stop`/`restart`/`run`/`status` for every service app and `install-dev`/`install-prod`/`upgrade`/`rollback` for every service + package app, with nested-workspace apps excluded from both groups; `scripts/build.sh` is `exec funbuild build`, not a hand-rolled per-app loop.
- Every app's category (service / package / nested workspace) was decided by the test in [references/rules.md](references/rules.md), never guessed from its name suffix, and the code (`resolve_service_app`/`resolve_release_app`) agrees with the README app table.
- `docs/` exists and follows the document-bundle-standard layout (`NNN-kebab-case.md`, each bundle starting from `001-overview.md`); no `apps/<name>` repo has grown its own parallel `docs/` bundle unless it's a nested `-dev` workspace or a dedicated `<product>-doc` repo.
- `bash -n` any touched shell script.
- If you changed an app repo, confirm its own commit is pushed before you commit the dev repo's pointer update.
- If you touched `<product>-web`, confirm it still proxies backend-facing paths to `<product>-api` (not just serving static assets) and that dev-time tooling proxies the same backend URL as the production server.
