# FarFarFun Service Governance

面向 Codex 的服务工程规范 Skill 集合：发布治理、Bash 生命周期脚本、多仓库子模块编排。

## Skills

| Skill | 能力 |
| --- | --- |
| [`service-release-governance`](skills/service-release-governance/SKILL.md) | 约束服务通过正式包发布、仓库安装和生产启动 |
| [`bash-service-guide`](skills/bash-service-guide/SKILL.md) | 统一 Bash 服务生命周期脚本及开发、生产运行边界 |
| [`submodule-workspace-governance`](skills/submodule-workspace-governance/SKILL.md) | 搭建和审计以 `apps/` 下 Git 子模块聚合多个独立仓库应用的 `<product>-dev` 编排仓库 |

这三个 skill 互相引用最密集（`submodule-workspace-governance` 依赖前两者定义 app 仓库应满足的发布和启动规范），放在同一仓库内可以保持相对链接不失效。

## Requirements

- Codex
- Git
- Bash

## Install

```bash
git clone https://github.com/farfarfun-skill/service-governance.git
cd service-governance

mkdir -p "${CODEX_HOME:-$HOME/.codex}/skills"
for skill in skills/*; do
  ln -s "$(pwd)/$skill" "${CODEX_HOME:-$HOME/.codex}/skills/$(basename "$skill")"
done
```

链接命令在同名 Skill 已存在时会失败，不会覆盖现有安装。更新仓库后，符号链接会继续指向最新工作树。

## Quick Start

在 Codex 中直接说明要使用的 Skill：

```text
Use $service-release-governance to review how this service is packaged, installed, and started in production.
Use $bash-service-guide to design or audit scripts/setup.sh and per-service lifecycle scripts.
Use $submodule-workspace-governance to scaffold or audit this <product>-dev repository's apps/ submodules and scripts/ layout.
```

完整项目背景见 [`docs/project/project-overview.md`](docs/project/project-overview.md)。
