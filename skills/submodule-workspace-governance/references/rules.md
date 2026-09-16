# Submodule Workspace Rules

Expanded invariants and a review checklist for `<product>-dev` submodule workspaces. Use this when the workspace has grown past two apps, when reviewing someone else's dev-repo change, or when SKILL.md's summary isn't enough to resolve an edge case.

## Repo Boundary Rules

- The dev repo's own tracked files are limited to: `README.md`, `.gitmodules`, `scripts/`, optional `docs/`, optional `logs/`, and standard repo metadata (`LICENSE`, `.gitignore`, CI config). No `src/`, no app-specific config, no application dependencies (`package.json`, `pyproject.toml`) at the dev-repo root.
- If a file seems like it belongs to "the product" but not to any one app specifically (e.g. a shared architecture doc, a cross-app changelog), put it under `docs/` in the dev repo — do not force it into one app's repo just because that app happens to exist first.
- A submodule directory (`apps/<name>`) is never `.gitignore`d and never has its contents modified by the dev repo's own commits — only `git submodule add`/`update`/pointer commits touch it from the dev-repo side.

## Naming Rules

- Dev repo: `<product>-dev`.
- Backend/core service repo: `<product>` (no suffix) when there is exactly one canonical backend; otherwise `<product>-<role>` for each (`<product>-api`, `<product>-worker`).
- Frontend repo: `<product>-web`.
- Submodule path under `apps/` matches the repo name exactly — never rename the path to something shorter or different from the remote repo it tracks.

## Script Rules

- `scripts/init.sh` must work from a plain `git clone` (no `--recurse-submodules`) with zero arguments and zero prompts.
- `scripts/build.sh` must be safe to re-run: switching each app to its tracked branch before building means a dirty or detached submodule checkout doesn't silently build the wrong commit.
- `scripts/build.sh` builds every app before running `funbuild push` once at the end — do not push after each individual app, since that produces a dev-repo commit per app instead of one atomic "these versions ship together" commit.
- Keep `scripts/all.sh` and `scripts/setup.sh` out of the workspace until there's a concrete recurring need; an unused `all` action is dead code that will drift from what the apps actually support the moment one app's interface changes.
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
- [ ] `all.sh`/`setup.sh`, if present, are still exercised by an actual documented use case — remove them if they've gone stale.
