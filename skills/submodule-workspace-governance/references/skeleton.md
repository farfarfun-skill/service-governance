# Submodule Workspace Skeleton

Starter layout for a `<product>-dev` orchestration repo with two apps (backend `<product>-api` and frontend `<product>-web`), both CLI-bearing. Match repo names, app names, and script contents to the target product instead of copying placeholders literally. See the end of this file for adding a non-CLI app (core library or plugin) once the product grows past two repos.

## Layout

```
<product>-dev/
├── README.md
├── .gitmodules
├── apps/
│   ├── <product>-api/      # submodule -> https://github.com/<org>/<product>-api.git
│   └── <product>-web/      # submodule -> https://github.com/<org>/<product>-web.git
└── scripts/
    ├── init.sh
    ├── build.sh
    ├── setup.sh             # optional, add once there's a CLI-bearing app
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
set -e

git -C apps/<product>-api switch master
git -C apps/<product>-web switch master

cd apps/<product>-api
funbuild build

cd ../..
cd apps/<product>-web
funbuild build

cd ../..

funbuild push
```

`funbuild push` runs at the dev-repo level after both apps build, so the resulting submodule pointer bump is committed and pushed as part of the same operation. See [Service Release Governance](../../service-release-governance/SKILL.md) for what `funbuild build`/`funbuild install` do per ecosystem.

## scripts/setup.sh (optional, add once there's a CLI-bearing app)

Usage: `scripts/setup.sh <action> <target>`, where `<target>` is `api`, `web`, or `all`. `all` means something different per action group — CLI-bearing apps only for service actions, every app under `apps/` for `build`. Do not add this until the workspace has at least one `-web`/`-api` app; a workspace of core-library/plugin apps only never needs it.

```sh
#!/bin/sh
set -e

cd "$(dirname "$0")/.."

# CLI-bearing apps only: short alias -> submodule path. Only these implement
# the Entrypoint Contract and their own scripts/setup.sh.
resolve_cli_app() {
  case "$1" in
    api) echo "apps/<product>-api" ;;
    web) echo "apps/<product>-web" ;;
    *) echo "" ;;
  esac
}
cli_apps="api web"

# Every submodule under apps/, CLI-bearing or not (core library + plugins).
all_app_paths() {
  git submodule status | awk '{print $2}'
}

usage() {
  echo "usage: scripts/setup.sh <start|stop|restart|run|status|install|publish> <api|web|all>" >&2
  echo "       scripts/setup.sh build <api|web|all|apps/<name>>" >&2
  exit 1
}

action="$1"
target="$2"
[ -n "$action" ] && [ -n "$target" ] || usage

case "$action" in
  start|stop|restart|run|status|install|publish)
    if [ "$target" = "all" ]; then
      apps="$cli_apps"
    else
      apps="$target"
    fi
    for app in $apps; do
      path="$(resolve_cli_app "$app")"
      [ -n "$path" ] || { echo "not a CLI-bearing app: $app" >&2; exit 1; }
      echo "== $app: $action =="
      (cd "$path" && ./scripts/setup.sh "$action")
    done
    ;;
  build)
    if [ "$target" = "all" ]; then
      paths="$(all_app_paths)"
    else
      path="$(resolve_cli_app "$target")"
      [ -n "$path" ] || path="apps/$target"
      paths="$path"
    fi
    for path in $paths; do
      echo "== $path: build =="
      git -C "$path" switch master
      (cd "$path" && funbuild build)
    done
    funbuild push
    ;;
  *)
    usage
    ;;
esac
```

Service actions delegate straight into the target app's own `scripts/setup.sh` (its `bash-service-guide`-compliant dispatcher) instead of reimplementing PID/port handling here. `build` runs directly against each resolved app path and always finishes with a single `funbuild push`, matching `scripts/build.sh`'s discipline above.

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

| 目录 | 项目 | 说明 |
| --- | --- | --- |
| `apps/<product>-api` | [<product>-api](https://github.com/<org>/<product>-api) | <one-line purpose> |
| `apps/<product>-web` | [<product>-web](https://github.com/<org>/<product>-web) | <one-line purpose> |

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

## Adding a non-CLI app (core library or plugin)

Once the product grows past the frontend/backend pair — a core library the backend depends on, or a plugin/driver like a specific cloud storage integration — wire it in the same way but skip the service scripts:

```bash
git submodule add https://github.com/<org>/<product>.git apps/<product>
git submodule add https://github.com/<org>/<product>-aliyun.git apps/<product>-aliyun
```

- They are automatically included by `scripts/setup.sh build all` (or `scripts/build.sh`, if it iterates every submodule) since `all_app_paths()` reads every entry from `.gitmodules` — no separate wiring needed for the build path.
- Never add them as a `<target>` alias for service actions (`start`/`stop`/`restart`/`run`/`status`/`install`/`publish`) in `scripts/setup.sh` — `resolve_cli_app` should only ever map `api`/`web`, so passing one of these apps to a service action fails loudly instead of silently no-op'ing.
- Add both to the README's app table like any other submodule; just note in the one-line purpose that it's a library/plugin, not a service (e.g. "Core domain library, consumed by `<product>-api`").
- Do not scaffold a per-app `scripts/setup.sh`-style start/stop CLI for these apps, and do not hold them to the Entrypoint Contract in review — see [references/rules.md](rules.md)'s CLI Entrypoint Requirement section.
