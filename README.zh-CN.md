# Quadlet Migrator

[English](./README.md) | [简体中文](./README.zh-CN.md)

一个 skill，用来将 `docker run` 命令和 Docker Compose 风格部署迁移为可维护的 Podman Quadlet 单元。

## 功能概览

- 将 `docker run` 和 Compose 风格输入转换为面向 Quadlet 的设计
- 帮助在 `.container`、`.pod`、`.network`、`.volume`、`.build` 之间做选择
- 在合适时保留 `.env` / `env_file` 工作流
- 将大型 env 模板归纳为少量高影响部署问题
- 说明 rootless / rootful 放置路径、部署说明与验证步骤

## 设计原则

- 优先选择满足需求的最轻运行模式
- 将规划、审阅、生成拆成明确阶段
- 不虚构部署相关取值
- 对有损映射保持显式说明
- 优先输出可维护的结果，而不是机械一比一转换

## 运行模式

- `advice`：解释映射方式或审查输入，不写最终产物
- `design`：执行 planning 和 finalize 审阅，但在生成可运行产物前停止
- `generate`：在 planning 和 finalize 审阅通过后，生成已批准的可运行产物

## 文档说明

- `SKILL.md`：运行模式、工作流和高层规则
- `references/compose-mapping.md`：Compose 字段映射与拓扑决策
- `references/env-strategy.md`：env 处理与 secret 默认策略
- `references/github-repo-intake.md`：仓库发现与 canonical 输入选择
- `references/deployment-notes.md`：部署说明
- `references/validation.md`：验证与排障

## 限制

本 skill 不保证 Docker Compose 语义与 Podman Quadlet 语义完全等价。

## 许可证

MIT
