# Submodule Workspace Rules

Expanded invariants and a review checklist for `<product>-dev` submodule workspaces. Use this when the workspace has grown past two apps, when reviewing someone else's dev-repo change, or when SKILL.md's summary isn't enough to resolve an edge case.

## Repo Boundary Rules

- The dev repo's own tracked files are limited to: `README.md`, `.gitmodules`, `scripts/`, `docs/` (required, see below), optional `logs/`, and standard repo metadata (`LICENSE`, `.gitignore`, CI config). No `src/`, no app-specific config, no application dependencies (`package.json`, `pyproject.toml`) at the dev-repo root.
- `docs/` is required, not optional, and follows `project-structure-governance`'s document-bundle-standard layout (`docs/product|design|development|testing|retrospective|release/<feature-slug>/`, each bundle numbered from `001-overview.md`). If a file seems like it belongs to "the product" but not to any one app specifically (e.g. a shared architecture doc, a cross-app changelog, a coordinated-release writeup), put it under `docs/` in the dev repo — do not force it into one app's repo just because that app happens to exist first.
- App repos under `apps/<name>` do not maintain their own `docs/` bundle — that would duplicate or fragment what belongs in the dev repo's `docs/`. Two exceptions: an app that is itself a nested `<something>-dev` submodule workspace owns its own `docs/` recursively; or the workspace has a dedicated `apps/<product>-doc` repo whose whole purpose is documentation. An app's own `README.md` (purpose, install, per-app usage) is unaffected by this rule — that always stays in the app repo.
- A submodule directory (`apps/<name>`) is never `.gitignore`d and never has its contents modified by the dev repo's own commits — only `git submodule add`/`update`/pointer commits touch it from the dev-repo side.

## Naming Rules

- Dev repo: `<product>-dev`.
- Core library repo: `<product>` (no suffix) — holds the actual domain logic that other apps depend on as an installed package. Small products without a separable library can fold this straight into `<product>-api` instead of forcing an empty split.
- Backend/server repo: `<product>-api` — a thin service that depends on `<product>` (and any plugin repos) and exposes it to `<product>-web`. Prefer `-api` over `-server`/`-backend`/`-svc`: once the service is running it genuinely serves an API to other consumers, not just the paired frontend, so the name describes what it does rather than just "a process exists."
- Frontend repo: `<product>-web`.
- Plugin/driver/SDK repo: `<product>-<capability>` (e.g. `<product>-aliyun`, `<product>-oss`) — a focused integration consumed by `<product>` or `<product>-api`, or published standalone for third parties. Never reuse this pattern for another frontend or backend pairing — those always stay `-web` / `-api`.
- Nested workspace app: `<something>-dev` under `apps/` — an app that is itself another `<product>-dev` submodule workspace, recursively governed by this same skill. This is the one suffix that is also a reliable structural signal (it has its own `apps/` + `scripts/setup.sh`), unlike the service/package split below.
- Submodule path under `apps/` matches the repo name exactly — never rename the path to something shorter or different from the remote repo it tracks.
- `-api`/`-web`/`-<capability>` are naming conventions for the common case, not the classification mechanism — see App Category & Lifecycle Requirement below for why the suffix alone never decides whether an app needs the CLI Entrypoint Contract.

## App Category & Lifecycle Requirement

Every app under `apps/` falls into exactly one of three categories. The category — never the repo's name suffix — decides which `scripts/setup.sh` actions it must support and how the dev repo's own `scripts/setup.sh` dispatches to it.

- **Service app** — something starts it as a long-running process and talks to it (HTTP, a socket, a queue, ...). Must satisfy [Service Release Governance](../../service-release-governance/SKILL.md)'s Entrypoint Contract and [Bash Service Guide](../../bash-service-guide/SKILL.md)'s `scripts/setup.sh` pattern in full: both service actions (`start`/`stop`/`restart`/`run`/`status`) and release actions (`install-dev`/`install-prod`/`upgrade`/`rollback`/`publish`). `-api` and `-web` are the conventional names for the two most common service apps, but any app that passes the test below is a service app regardless of name — a queue worker or scheduler counts just as much.
- **Package app** — only ever imported or `pip install`/`npm install`ed as a dependency; nothing starts it as its own process. Still must satisfy service-release-governance's release actions (`install-dev`/`install-prod`/`upgrade`/`rollback`/`publish`) plus `build`, since it's still a versioned artifact someone installs — it just has no `server` subcommands and no service actions. The core library (`<product>`) and plugin/driver repos (`<product>-<capability>`) are the conventional examples.
- **Nested workspace app** — an `apps/<name>` submodule that is itself a `<something>-dev` submodule workspace, recursively governed by this same skill. Out of scope for this dev repo's own action dispatch: don't wire it into `resolve_service_app`/`resolve_release_app`, and don't hold it to either action group here. Bump its submodule pointer like any other app; to act on what's inside it, `cd` into it and run its own `scripts/setup.sh` directly — it owns and dispatches its own three-way classification independently.

Test to apply per app (service vs. package — a nested workspace app is identified structurally, by containing its own `apps/`+`scripts/setup.sh`, not by this test): "does something start this as a long-running process it then talks to?" Yes → service app. Only ever installed as a dependency → package app.

Do not infer the category from the repo's name suffix. `-api`/`-web` are conventions, not proof: a `-web` app that ships as static files with no local server process, or a `-api` deployed as a set of serverless functions with nothing to `start`/`stop` locally, is a package app despite the name; conversely a differently-named app (`-worker`, `-consumer`, `-scheduler`) is a service app the moment something runs it as a process. Settle the classification explicitly in two places that must agree, not by guessing from the string:

1. Code: whether `scripts/setup.sh` wires the app's alias into `resolve_service_app()` (service) or only into `resolve_release_app()`/`all_app_paths()` (package) — see [skeleton.md](skeleton.md).
2. Docs: the app's row in the dev repo README's app table states its category outright ("service" / "package" / "nested workspace"), not just a free-text description.

A review that finds the code and the README disagreeing on an app's category is a defect, not a matter of judgment.

Example: a `fundrive-dev` workspace with `apps/fundrive` (core library, package), `apps/fundrive-api` (backend, service), `apps/fundrive-web` (frontend, service), `apps/fundrive-aliyun` (storage-driver plugin, package) — plus, if the product later grows a background indexer, `apps/fundrive-indexer` (service, despite not being named `-api`/`-web`) — holds `fundrive-api`, `fundrive-web`, and `fundrive-indexer` to the full service+release action set; `fundrive` and `fundrive-aliyun` get release actions and `build` only.

## Web App Proxy Requirement

- `<product>-web`'s own runtime server does two jobs, not one: serve the built static assets under its base path, and reverse-proxy every backend-facing path (`/api/**`, plus anything else `<product>-api` exposes to the browser, e.g. `/healthz`) straight through to `<product>-api`'s base URL.
- This is not optional once `-web` and `-api` deploy to different origins/ports: without the proxy, the browser calls `<product>-api` directly and hits CORS, because `-api` is not expected to run CORS middleware — the paired `-web` proxy is what keeps every browser request same-origin. Do not "fix" this by adding CORS headers to `-api` instead; that reopens the backend to arbitrary origins and duplicates a concern the proxy already owns.
- Resolve the backend base URL with the same precedence the Entrypoint Contract already uses elsewhere: an explicit CLI flag (e.g. `--backend`) overrides a config-file field, which overrides an env var (e.g. `<PRODUCT>_API_BASE_URL`), which falls back to a hardcoded local-dev default (`http://127.0.0.1:<api-port>`).
- Point dev-time tooling (Vite/webpack devServer proxy, etc.) at the same env var so a local dev session (`pnpm dev`, etc.) proxies exactly like the packaged CLI's production server — no separate, drifting proxy config for dev vs. prod.
- Routing must be unambiguous: proxy paths never fall through to the SPA `index.html` fallback, and static-asset paths never get proxied.
- Worked example: `funflix-web`'s `server/app.js` (routes `/` → redirect, `/api/**` + `/healthz` → proxy, `/web/**` → static+SPA fallback) and `server/proxy.js` (the actual reverse proxy, plain `node:http`/`node:https`, no dependency) implement this; its `vite.config.ts` dev-time `server.proxy` block targets the same `FUNFLIX_API_BASE_URL` env var the production CLI reads via `--backend`/config/env.

## Script Rules

- `scripts/init.sh` must work from a plain `git clone` (no `--recurse-submodules`) with zero arguments and zero prompts.
- `scripts/build.sh` must be safe to re-run: switching each app to its tracked branch before building means a dirty or detached submodule checkout doesn't silently build the wrong commit.
- `scripts/build.sh` builds every app before running `funbuild push` once at the end — do not push after each individual app, since that produces a dev-repo commit per app instead of one atomic "these versions ship together" commit.
- Keep `scripts/all.sh` out of the workspace until there's a concrete recurring need for a plain non-service batch loop; an unused `all` action is dead code that will drift from what the apps actually support the moment one app's interface changes.
- `scripts/setup.sh` is required and takes `<action> <target>`, mirroring `bash-service-guide`'s own `action -> service` resolution one level up:
  - `<target>` is a short alias resolved to an `apps/<name>` path in one place, or `all`. The alias name is a convention (`api`, `web`, ...), not the classification mechanism — see App Category & Lifecycle Requirement above.
  - Actions split into two groups, each scoped to `all` by app category rather than by name:
    - **Service actions** (`start`, `stop`, `restart`, `run`, `status`): `all` expands to service apps only. Delegate each call to that app's own `scripts/setup.sh <action>` — never reimplement PID/port/process handling at the dev-repo level.
    - **Release actions** (`install-dev`, `install-prod`, `upgrade`, `rollback`): `all` expands to every service app and every package app — a core-library or plugin app still needs these even though it has no service actions. `build` and `publish` are not `setup.sh` actions at all — `funbuild build`, run from the dev repo root, owns both across every app under `apps/` on its own.
  - Both groups exclude nested-workspace apps; those get their pointer bumped like any other submodule but are never dispatched into by this script.
  - An action against a target outside its group's scope (a service action against a package or nested-workspace app) is a usage error, not a silent no-op — fail loudly with a clear message.
- Any script that touches more than one app must state the order apps are processed in and what happens on partial failure (stop at first failure, matching `set -e`, unless the workspace has an explicit reason to continue).

## Submodule Pointer Discipline

- Before touching an app submodule, check `git submodule status` to confirm you're not about to build on top of an uninitialized (`-` prefix) or already-modified (`+` prefix) submodule you didn't expect.
- A dev-repo commit that changes `apps/<name>` must only ever bump that submodule's recorded commit — never hand-edit tracked files inside `apps/<name>` and commit that as part of the dev repo (that change belongs in the app repo, with its own commit and push, first).
- When multiple apps change together for a coordinated release, stage all of their pointer bumps into one dev-repo commit (`git add apps/<a> apps/<b> && git commit`) rather than one dev-repo commit per app, so the dev repo's history reads as "release N points at these commits" rather than a stream of unrelated single-app bumps.
- Never force-push the dev repo to rewrite a pointer history other contributors may have already built on top of.

## Review Checklist

When reviewing a change to a `<product>-dev` repo, confirm:

- [ ] No application source was added outside `apps/`.
- [ ] `.gitmodules` only contains entries added via `git submodule add`/`sync`, with path matching the repo name.
- [ ] `scripts/init.sh` still works from a fresh plain clone.
- [ ] `scripts/build.sh` (if touched) still switches each app to its tracked branch before building, and still runs `funbuild push` exactly once at the end.
- [ ] Every submodule pointer change has a corresponding already-pushed commit in that app's own repo.
- [ ] The README's app table still matches `.gitmodules` exactly (same set of apps, same paths).
- [ ] `scripts/setup.sh` exists (required, not optional) and resolves `all` per the two-way split (service apps only for service actions; service + package apps for release actions; nested-workspace apps excluded from both), rejecting an action against a target outside its group instead of silently skipping it. `build`/`publish` are not `setup.sh` actions — `funbuild build` owns them.
- [ ] `all.sh`, if present, is still exercised by an actual documented use case — remove it if it's gone stale.
- [ ] Every app's category (service / package / nested workspace) was decided by the test in App Category & Lifecycle Requirement, never inferred from its name suffix — and the code (`resolve_service_app`/`resolve_release_app`) and the README app table agree on that category for every app.
- [ ] `-web`'s own server reverse-proxies backend-facing paths to `-api` (not just serving static assets), the backend URL resolution follows the same flag/config/env precedence as the rest of the Entrypoint Contract, and dev tooling proxies the same target — CORS was not added to `-api` as a substitute.
- [ ] `docs/` exists, follows the document-bundle-standard layout, and no `apps/<name>` repo has grown a parallel `docs/` bundle of its own (unless it's a nested `-dev` workspace or a dedicated `<product>-doc` repo).
