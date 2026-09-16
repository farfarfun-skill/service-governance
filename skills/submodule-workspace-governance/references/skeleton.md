# Submodule Workspace Skeleton

Starter layout for a `<product>-dev` orchestration repo with two apps (backend `<product>` and frontend `<product>-web`). Match repo names, app names, and script contents to the target product instead of copying placeholders literally.

## Layout

```
<product>-dev/
├── README.md
├── .gitmodules
├── apps/
│   ├── <product>/          # submodule -> https://github.com/<org>/<product>.git
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

git submodule add https://github.com/<org>/<product>.git apps/<product>
git submodule add https://github.com/<org>/<product>-web.git apps/<product>-web
```

This generates `.gitmodules`:

```ini
[submodule "apps/<product>"]
	path = apps/<product>
	url = https://github.com/<org>/<product>.git
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

git -C apps/<product> switch master
git -C apps/<product>-web switch master

cd apps/<product>
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

apps="<product> <product>-web"
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
| `apps/<product>` | [<product>](https://github.com/<org>/<product>) | <one-line purpose> |
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
git add apps/<product> apps/<product>-web
\`\`\`

更新后的子模块提交由当前仓库记录，需要随父仓库一起提交。

## 构建

安装并配置好 \`funbuild\` 后执行：

\`\`\`bash
bash scripts/build.sh
\`\`\`
```
