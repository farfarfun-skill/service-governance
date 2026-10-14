# FarFarFun Service Governance

面向 Codex 的服务工程规范 Skill 集合：发布治理、Bash 生命周期脚本、多仓库子模块编排。

> 跨仓库（含 `lang-spec-hub` 等）的完整 Skill 路由一览见 [`skill-router`](https://github.com/farfarfun-skill/skill-router)。

## Skills

| Skill | 能力 |
| --- | --- |
| [`service-release-governance`](skills/service-release-governance/SKILL.md) | 约束服务通过正式包发布、仓库安装和生产启动 |
| [`bash-service-guide`](skills/bash-service-guide/SKILL.md) | 统一 Bash 服务生命周期脚本及开发、生产运行边界 |
| [`submodule-workspace-governance`](skills/submodule-workspace-governance/SKILL.md) | 搭建和审计以 `apps/` 下 Git 子模块聚合多个独立仓库应用的 `<product>-dev` 编排仓库 |
| [`python-development-standards`](skills/python-development-standards/SKILL.md) | Python 代码、测试、项目配置及 farfarfun 工具复用规范 |
| [`java-development-standards`](skills/java-development-standards/SKILL.md) | Java 代码、测试、Maven/Gradle 项目的实现和审查 |

前三个 skill 互相引用最密集（`submodule-workspace-governance` 依赖前两者定义 app 仓库应满足的发布和启动规范），放在同一仓库内可以保持相对链接不失效。`python-development-standards`/`java-development-standards` 从 [`lang-spec-hub`](https://gitee.com/farfarfun-skills/farfarfun-skill--lang-spec-hub) 同步而来，供本仓库内需要具体语言编码规范的场景使用。

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
Use $python-development-standards to implement or review this Python change.
Use $java-development-standards to implement or review this Java change.
```

完整项目背景见 [`docs/project/project-overview.md`](docs/project/project-overview.md)。

## 关于组织

本仓库属于 [farfarfun-skill](https://github.com/farfarfun-skill) —— [farfarfun](https://github.com/farfarfun) 旗下的 AI Agent Skill 产品线子组织，也是全部自研 Skill 的源头。组织内各仓库分别聚焦一个治理域，把原本依赖人工评审的规范，转化为 Agent 可直接调用的确定性检查器与工作流；issue、PR 与规范变更均在此处提交。

本仓库在其中负责服务工程规范：发布治理、Bash 生命周期脚本与多仓库子模块编排（其中的语言规范副本同步自 [`lang-spec-hub`](https://github.com/farfarfun-skill/lang-spec-hub)）。要确认某个任务该用哪个 Skill，先查 [`skill-router`](https://github.com/farfarfun-skill/skill-router)；需要批量安装或通过 Gitee 访问，请前往镜像组织 [farfarfun-skills](https://github.com/farfarfun-skills)。
