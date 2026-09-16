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
    └── all.sh               # optional, add only when needed
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

## scripts/all.sh (optional)

Only add this once the workspace needs one command fanned out across every app — for example checking status or pulling latest across all of them:

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

- Add both to `scripts/build.sh` alongside the CLI apps if they need `funbuild build`/`funbuild install` to publish a package — but never add them to `scripts/all.sh`'s service-lifecycle actions (`start`/`stop`/`status`) since nothing runs them as a process.
- Add both to the README's app table like any other submodule; just note in the one-line purpose that it's a library/plugin, not a service (e.g. "Core domain library, consumed by `<product>-api`").
- Do not scaffold `scripts/setup.sh`-style start/stop commands for these apps, and do not hold them to the Entrypoint Contract in review — see [references/rules.md](rules.md)'s CLI Entrypoint Requirement section.
