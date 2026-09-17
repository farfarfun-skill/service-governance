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
- Submodule path under `apps/` matches the repo name exactly — never rename the path to something shorter or different from the remote repo it tracks.

## CLI Entrypoint Requirement

- Only the `-web` (frontend) and `-api` (backend/server) apps must satisfy [Service Release Governance](../../service-release-governance/SKILL.md)'s Entrypoint Contract and, once they have a runnable service, [Bash Service Guide](../../bash-service-guide/SKILL.md)'s `scripts/setup.sh` pattern.
- The core library repo (`<product>`) and plugin/driver repos (`<product>-<capability>`) are exempt — they ship as installable packages consumed by `<product>-api` or third parties, not as long-running services. Forcing a `server start` CLI onto a repo nobody starts as a process just adds unused surface.
- Test to apply per app: "does something start this as a long-running process it then talks to?" Yes → Entrypoint Contract required. Only ever imported or `pip install`/`npm install`ed as a dependency → exempt.
- Example: a `fundrive-dev` workspace with `apps/fundrive` (core library), `apps/fundrive-api` (backend), `apps/fundrive-web` (frontend), `apps/fundrive-aliyun` (storage-driver plugin) only holds `fundrive-api` and `fundrive-web` to the CLI requirement; `fundrive` and `fundrive-aliyun` are installed, not started.

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
  - `<target>` is a short alias (`api`, `web`, ...) resolved to `apps/<product>-api` / `apps/<product>-web` in one place, or `all`.
  - For service actions (`start`, `stop`, `restart`, `run`, `status`, `install-dev`, `install-prod`, `upgrade`, `rollback`, `publish`), `all` expands to CLI-bearing apps only (`api` + `web`) and each call delegates to that app's own `scripts/setup.sh <action>` — never reimplement PID/port/process handling at the dev-repo level.
  - For `build`, `all` expands to every submodule under `apps/`, CLI-bearing or not, since core-library and plugin apps still need `funbuild build`/`funbuild install` to be published even though nothing starts them as a process. Run `funbuild push` once at the end regardless of scope.
  - A service action against a target that resolves to a non-CLI app (core library or plugin) is a usage error, not a silent no-op — fail loudly with a clear message.
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
- [ ] `scripts/setup.sh` exists (required, not optional) and resolves `all` differently per action group (CLI apps only for service actions, every app under `apps/` for `build`), rejecting service actions against non-CLI targets instead of silently skipping them.
- [ ] `all.sh`, if present, is still exercised by an actual documented use case — remove it if it's gone stale.
- [ ] The CLI/service Entrypoint Contract was applied to the `-web`/`-api` apps only — not skipped for either of them, and not forced onto a core-library or plugin app that nothing starts as a process.
- [ ] `-web`'s own server reverse-proxies backend-facing paths to `-api` (not just serving static assets), the backend URL resolution follows the same flag/config/env precedence as the rest of the Entrypoint Contract, and dev tooling proxies the same target — CORS was not added to `-api` as a substitute.
- [ ] `docs/` exists, follows the document-bundle-standard layout, and no `apps/<name>` repo has grown a parallel `docs/` bundle of its own (unless it's a nested `-dev` workspace or a dedicated `<product>-doc` repo).
