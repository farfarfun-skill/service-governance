# 项目说明

> 固定路径：`docs/project/project-overview.md`

## 基本信息

- 项目名称：FarFarFun Service Governance
- 项目简介：面向 Codex 的服务工程规范 Skill 集合
- 项目负责人：FarFarFun
- 代码仓库：https://github.com/farfarfun-skill/service-governance

## 应用清单

<!-- project-structure:applications:start -->
| 应用 | 路径 | 技术 Profile | Owner | 用途 |
| --- | --- | --- | --- | --- |
| 无（单应用仓库） | - | - | - | - |
<!-- project-structure:applications:end -->

## 项目目标

- 约束服务通过正式包发布、仓库安装和生产启动，禁止生产直接运行开发源码。
- 统一 Bash 服务生命周期脚本的行为边界（`start`/`run`、`install-dev`/`install-prod`/`upgrade`/`rollback`/`publish`）。
- 规范以 `apps/` 下 Git 子模块聚合多个独立仓库应用的 `<product>-dev` 编排仓库的搭建与审计。

## 项目范围

### 范围内

- npm / Python 服务的构建、安装、生产启动规范。
- `scripts/setup.sh` 及各服务生命周期脚本的设计、审计、重构。
- `<product>-dev` 编排仓库的 `apps/` 子模块与 `scripts/` 布局。

### 范围外

- 通用项目生命周期治理（见 `farfarfun-skill/project-manager` 仓库）。
- Paperclip 平台专属治理（见 `farfarfun-skill/paperclip-governance` 仓库）。
- 代替产品、技术、测试或发布负责人作最终业务决策。

## 重要约定

- `skills/` 是本仓库的核心交付区域，每个直接子目录对应一个独立 Skill。
- 三个 skill 均为纯文档型（无 `scripts/`/`tests/`），互相之间用仓库内相对链接引用。
- 所有变更必须通过 Markdown 链接检查和 Skill 元数据校验。

## 技术与运行环境

- 主要技术栈：Markdown、YAML、GitHub Actions
- 开发环境：支持 Git 的本地环境
- 生产环境：由 Codex 从已安装的 Skill 目录读取并执行

## 相关文档

- 项目入口：`README.md`
- 项目结构索引：`.project-structure.json`
- 服务发布治理：`skills/service-release-governance/SKILL.md`
- Bash 生命周期脚本：`skills/bash-service-guide/SKILL.md`
- 子模块编排治理：`skills/submodule-workspace-governance/SKILL.md`

## 更新记录

| 日期 | 修改人 | 变更摘要 |
| --- | --- | --- |
| 2026-09-16 | FarFarFun | 从 `farfarfun--project-manager` 拆分为独立仓库，新增 `submodule-workspace-governance`，保留原始提交历史 |
