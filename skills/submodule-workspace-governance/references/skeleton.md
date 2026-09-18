# Submodule Workspace Skeleton

Starter layout for a `<product>-dev` orchestration repo with two apps (backend `<product>-api` and frontend `<product>-web`), both service apps. Match repo names, app names, and script contents to the target product instead of copying placeholders literally. Apps fall into three categories — service, package, nested workspace — decided by what they do, not by their name; see [references/rules.md](rules.md)'s App Category & Lifecycle Requirement. See the end of this file for adding a package app (core library or plugin) or a nested-workspace app once the product grows past two repos.

## Layout

```
<product>-dev/
├── README.md
├── .gitmodules
├── apps/
│   ├── <product>-api/      # submodule -> https://github.com/<org>/<product>-api.git
│   └── <product>-web/      # submodule -> https://github.com/<org>/<product>-web.git
├── docs/                    # required; document-bundle-standard layout, see below
└── scripts/
    ├── init.sh
    ├── build.sh
    ├── setup.sh             # required, alongside init.sh/build.sh
    ├── funbuild.toml        # required; shared version for every app under apps/
    └── all.sh                # optional, add only when needed
```

## Creating the submodules

```bash
git clone https://github.com/<org>/<product>-dev.git
cd <product>-dev

git submodule add https://github.com/<org>/<product>-api.git apps/<product>-api
git submodule add https://github.com/<org>/<product>-web.git apps/<product>-web
```

This generates `.gitmodules`:

```ini
[submodule "apps/<product>-api"]
	path = apps/<product>-api
	url = https://github.com/<org>/<product>-api.git
[submodule "apps/<product>-web"]
	path = apps/<product>-web
	url = https://github.com/<org>/<product>-web.git
```

## scripts/init.sh

```sh
git submodule init
git submodule update
```

## scripts/build.sh

```sh
#!/bin/sh
exec funbuild build
```

`funbuild build`, run at the dev-repo root, recognizes this layout (an `apps/` directory plus `scripts/funbuild.toml`) on its own: it bumps the shared version in `scripts/funbuild.toml`, builds/publishes every non-nested-workspace app under `apps/` with that same version, then commits/pushes/tags the dev repo once so the submodule pointer bumps land together. `build.sh` does not need to loop over apps itself. See [Service Release Governance](../../service-release-governance/SKILL.md) for what `funbuild build` does per ecosystem inside each app.

## scripts/funbuild.toml (required)

```toml
version = "1.0.0"
```

`funbuild build` reads and increments `version` so every app under `apps/` publishes under the same shared version number, instead of each app tracking its own. Commit the file; `funbuild` rewrites it in place as part of each build.

## scripts/setup.sh (required, alongside init.sh/build.sh)

Usage: `scripts/setup.sh <action> <target>`, where `<target>` is `api`, `web`, or `all`. `all` means something different per action group — service apps only for service actions, service + package apps for release actions, and nested-workspace apps excluded from both groups. Build and publish are not actions this script handles at all — `funbuild build`, run from the dev repo root, owns both across every app under `apps/` on its own (see `scripts/funbuild.toml` above). Every `<product>-dev` repo ships `setup.sh` from the start; even a workspace of package apps only still needs it for release actions (service actions simply have no alias to resolve until a service app is added).

A missing `<action>` or `<target>` falls back to a `gum choose` menu instead of erroring immediately — reuse `bash-service-guide`'s own `choose()` helper (see its [skeleton.md](../../bash-service-guide/references/skeleton.md)) rather than inventing a second interactive style at the dev-repo level. This needs Bash (`[[ ]]`, arrays), not POSIX `sh`, unlike the other cross-repo scripts.

```bash
#!/usr/bin/env bash
set -euo pipefail

ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"
cd "${ROOT}"

readonly -a SERVICE_ACTIONS=(start stop restart run status)
readonly -a RELEASE_ACTIONS=(install-dev install-prod upgrade rollback)
readonly -a ACTIONS=("${SERVICE_ACTIONS[@]}" "${RELEASE_ACTIONS[@]}")
readonly -a TARGETS=(api web all)

usage() {
  printf 'Usage: %s <start|stop|restart|run|status|install-dev> <api|web|all>\n' "${0##*/}" >&2
  printf '       %s <install-prod|upgrade> <api|web|all> [version]\n' "${0##*/}" >&2
  printf '       %s rollback <api|web|all> <version>\n' "${0##*/}" >&2
}

die() {
  printf 'error: %s\n' "$*" >&2
  exit 2
}

contains() {
  local needle="$1"
  shift
  local item
  for item in "$@"; do
    [[ "${item}" == "${needle}" ]] && return 0
  done
  return 1
}

choose() {
  command -v gum >/dev/null 2>&1 ||
    die "missing argument and gum is unavailable; run with explicit arguments"
  gum choose "$@"
}

# Service apps: something starts these as a long-running process. Only
# these accept service actions (start/stop/restart/run/status). Register an
# app here yourself — never infer this from its name (see rules.md's App
# Category & Lifecycle Requirement).
resolve_service_app() {
  case "$1" in
    api) printf '%s\n' "apps/<product>-api" ;;
    web) printf '%s\n' "apps/<product>-web" ;;
    *) return 1 ;;
  esac
}

# Package apps: installed as a dependency, never started as their own
# process (e.g. a core library or plugin). Empty in this two-app starter —
# add an entry here, not to resolve_service_app, per "Adding a package app"
# below.
resolve_package_app() {
  case "$1" in
    *) return 1 ;;
  esac
}

# Release actions (install-dev/install-prod/upgrade/rollback) apply to
# service apps and package apps alike. Nested-workspace apps resolve through
# neither this nor resolve_service_app — dispatching into one is its own
# scripts/setup.sh's job, not this one's.
resolve_release_app() {
  resolve_service_app "$1" 2>/dev/null || resolve_package_app "$1" 2>/dev/null
}

is_service_action() {
  contains "$1" "${SERVICE_ACTIONS[@]}"
}

dispatch_app_action() {
  local action="$1" target="$2" version="${3:-}" app path
  local -a apps
  if [[ "${target}" == "all" ]]; then
    if is_service_action "${action}"; then
      apps=(api web)  # every registered service app
    else
      apps=(api web)  # every registered service + package app; extend once
                       # resolve_package_app has entries, e.g. (api web core aliyun)
    fi
  else
    apps=("${target}")
  fi
  for app in "${apps[@]}"; do
    if is_service_action "${action}"; then
      path="$(resolve_service_app "${app}")" || die "${action} only applies to a service app, not: ${app}"
    else
      path="$(resolve_release_app "${app}")" || die "${action} does not apply to: ${app}"
    fi
    printf '== %s: %s ==\n' "${app}" "${action}"
    (cd "${path}" && ./scripts/setup.sh "${action}" ${version:+"${version}"})
  done
}

main() {
  local action="${1:-}"
  local target="${2:-}"
  local version="${3:-}"

  [[ -n "${action}" ]] || action="$(choose "${ACTIONS[@]}")"
  contains "${action}" "${ACTIONS[@]}" || {
    usage
    die "unknown action: ${action}"
  }

  [[ -n "${target}" ]] || target="$(choose "${TARGETS[@]}")"

  case "${action}" in
    rollback)
      [[ -n "${version}" ]] || die "rollback requires an explicit version: ${0##*/} rollback <api|web|all> <version>"
      contains "${target}" "${TARGETS[@]}" || {
        usage
        die "unknown target: ${target}"
      }
      dispatch_app_action "${action}" "${target}" "${version}"
      ;;
    *)
      contains "${target}" "${TARGETS[@]}" || {
        usage
        die "unknown target: ${target}"
      }
      dispatch_app_action "${action}" "${target}" "${version}"
      ;;
  esac
}

main "$@"
```

Service and release actions both delegate straight into the target app's own `scripts/setup.sh` (its `bash-service-guide`-compliant dispatcher) instead of reimplementing PID/port or package-install handling here; `dispatch_app_action` picks `resolve_service_app` or `resolve_release_app` based on which group the action belongs to, so a service action against a package app (or a nested-workspace app, which resolves through neither) fails loudly instead of silently doing nothing. It forwards an optional trailing `version` argument unchanged so `install-prod [version]`/`upgrade [version]`/`rollback <version>` reach the per-app script exactly as documented in [Bash Service Guide](../../bash-service-guide/SKILL.md); `rollback` fails loudly here if no version was supplied instead of falling through to a `gum choose` menu, since there is no interactive default for it. Only `main`'s two `choose()` calls are interactive — every other function still requires its arguments explicitly, so a fully-specified invocation (`scripts/setup.sh start api`) never touches `gum` and works the same in CI or a script.

## scripts/all.sh (optional)

Only add this once the workspace needs a plain, non-service batch loop across every app — for example checking `git status` or pulling latest across all of them. This is not an `<action> <target>` dispatcher like `setup.sh`; keep it to simple per-app git commands:

```sh
#!/bin/sh
set -e

apps="<product>-api <product>-web"
action="$1"

for app in $apps; do
  echo "== $app: $action =="
  git -C "apps/$app" "$action"
done
```

## README.md template

```markdown
# <product>-dev

<product> 联合开发仓库，通过 Git 子模块固定后端与 Web 界面的版本。

## 项目

| 目录 | 项目 | 类别 | 说明 |
| --- | --- | --- | --- |
| `apps/<product>-api` | [<product>-api](https://github.com/<org>/<product>-api) | service | <one-line purpose> |
| `apps/<product>-web` | [<product>-web](https://github.com/<org>/<product>-web) | service | <one-line purpose> |

具体的安装、配置和开发方式见各子项目 README。

## 获取代码

首次克隆时同时拉取子模块：

\`\`\`bash
git clone --recurse-submodules https://github.com/<org>/<product>-dev.git
cd <product>-dev
\`\`\`

已有仓库可执行：

\`\`\`bash
bash scripts/init.sh
\`\`\`

## 更新子模块

\`\`\`bash
git submodule update --remote
git add apps/<product>-api apps/<product>-web
\`\`\`

更新后的子模块提交由当前仓库记录，需要随父仓库一起提交。

## 构建

安装并配置好 \`funbuild\` 后执行：

\`\`\`bash
bash scripts/build.sh
\`\`\`
```

## docs/ (required)

The dev repo is where cross-app product documentation lives — layout follows `project-structure-governance`'s `document-bundle-standard.md` (a separate skill repo; see its `references/document-bundle-standard.md`) exactly, scoped to `<product>-dev` instead of a single app:

```text
docs/
├── product/<feature-slug>/
│   └── 001-overview.md, 002-..., ...
├── design/<feature-slug>/
│   └── 001-overview.md, ...
├── development/<feature-slug>/
│   └── 001-overview.md, ...          # cross-app architecture, API contracts between -api and -web, etc.
├── testing/<feature-slug>/
│   └── 001-overview.md, test-cases/, test-report/
├── retrospective/<feature-slug>/
│   └── 001-overview.md, ...
└── release/<date>-<issue-key>-<slug>/
    └── 001-overview.md, ...          # one directory per coordinated multi-app release
```

Every document package still opens with `001-overview.md` and uses the same `NNN-kebab-case.md` numbering the standard defines — do not invent a different scheme at the dev-repo level.

- Individual `apps/<name>` repos do not maintain their own `docs/` — a shared/cross-app doc (architecture, a changelog spanning `-api` and `-web`, a release retrospective) belongs in the dev repo's `docs/`, not duplicated or split across app repos. An app's own `README.md` (purpose, install, per-app usage) stays in that app repo; that is not a `docs/` bundle and is unaffected by this rule.
- Two exceptions, both still resolved by `project-structure-governance`, just one level down: an app that is itself a nested `<something>-dev` submodule workspace owns its own `docs/` the same way, recursively; or the workspace adds a dedicated `apps/<product>-doc` repo whose entire purpose is documentation (e.g. a docs site or knowledge base), in which case that repo owns `docs/`-bundle content instead of the parent dev repo. Do not create a `<product>-doc` app speculatively — only once product docs have outgrown what fits in the dev repo's own `docs/`.

## Adding a package app (core library or plugin)

Once the product grows past the frontend/backend pair — a core library the backend depends on, or a plugin/driver like a specific cloud storage integration — wire it in the same way but skip the service scripts:

```bash
git submodule add https://github.com/<org>/<product>.git apps/<product>
git submodule add https://github.com/<org>/<product>-aliyun.git apps/<product>-aliyun
```

- Register each one as a `resolve_package_app` case, e.g.:
  ```bash
  resolve_package_app() {
    case "$1" in
      core) printf '%s\n' "apps/<product>" ;;
      aliyun) printf '%s\n' "apps/<product>-aliyun" ;;
      *) return 1 ;;
    esac
  }
  ```
  and extend `dispatch_app_action`'s non-service `all` list to match (`apps=(api web core aliyun)`). This registration — not the repo's name — is what makes it a package app; do not skip it and rely on the name suffix alone.
- They are automatically built and published by `funbuild build` (`scripts/build.sh`) without any registration here — it walks every non-nested-workspace submodule under `apps/` on its own, so a package app just needs to exist under `apps/` to be included.
- Never add them to `resolve_service_app` — `resolve_service_app` should only ever map apps something actually starts as a process, so passing a package app to a service action fails loudly instead of silently no-op'ing.
- Add both to the README's app table like any other submodule with `package` in the 类别/category column; the one-line purpose can still add color (e.g. "Core domain library, consumed by `<product>-api`"), but the category itself must be the explicit label, not just implied by that text.
- Do not scaffold a per-app `scripts/setup.sh`-style start/stop CLI for these apps, and do not hold them to the service-action part of the Entrypoint Contract in review — see [references/rules.md](rules.md)'s App Category & Lifecycle Requirement section. They still need `install-dev`/`install-prod`/`upgrade`/`rollback`/`publish` support in their own `scripts/setup.sh`, just no `server` subcommands.

## Adding a nested workspace app

If one of the product's own pieces is itself split into further submodules — a large plugin suite, or a sub-product with its own backend/frontend pair — it can be wired in as its own `<something>-dev` submodule workspace instead of a single repo:

```bash
git submodule add https://github.com/<org>/<product>-plugins-dev.git apps/<product>-plugins-dev
```

- `funbuild build` skips it automatically — it detects a nested workspace structurally (an `apps/<name>` that itself has `apps/` + `scripts/setup.sh`) and never dispatches into it; this workspace's own `scripts/build.sh`/`scripts/setup.sh` build/release/serve it recursively, not this one's. No registration needed for the build path.
- Never register it in `resolve_service_app` or `resolve_package_app` — it isn't a single service or package, it's a whole other workspace with its own three-way classification underneath.
- To act on what's inside it, `cd apps/<product>-plugins-dev && ./scripts/setup.sh <action> <target>` directly, or add a thin passthrough action in this dev repo's own `scripts/all.sh` if that becomes a frequent need — never teach this dev repo's `setup.sh` to understand a second layer of aliases.
- Add it to the README's app table with `nested workspace` in the 类别/category column, and link to its own README for what it contains.
