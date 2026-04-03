# Quadlet Migrator

[English](./README.md) | [简体中文](./README.zh-CN.md)

一个 skill，用来将 `docker run` 命令和 Docker Compose 风格部署迁移为可维护的 Podman Quadlet 单元。

## 功能概览

- 将 `docker run` 和 Compose 风格输入转换为面向 Quadlet 的设计
- 默认先把生成产物写到当前目录，便于审查后再应用
- 帮助在 `.container`、`.pod`、`.network`、`.volume`、`.build` 之间做选择，并对多容器服务默认偏向 pod-first 拓扑
- 在合适时保留 `.env` / `env_file` 工作流
- 将大型 env 模板归纳为少量高影响部署问题
- 可生成辅助脚本，其中 `install.sh` 是规范的 apply 步骤，另可生成 `reload.sh`、`start.sh`、`stop.sh`、`restart.sh`
- 会识别并交付运行所需的 repo 内配套文件，例如挂载配置、初始化资源和辅助脚本
- 在声称结果可运行前，会检查环境变量完整性，而不是只做 env 摘要
- 鼓励在 finalize 与 execution 阶段显式使用 support files 与 env completeness 检查清单
- 说明 rootless / rootful 应用目标路径、部署说明与验证步骤

## 设计原则

- 优先选择满足需求的最轻运行模式
- 将规划、审阅、生成拆成明确阶段
- 不虚构部署相关取值
- 对有损映射保持显式说明
- 优先输出可维护的结果，而不是机械一比一转换
- 默认先输出到当前目录供审查，再执行安装
- 当 pod 分组已经能清晰表达意图时，优先采用 pod-first 拓扑而不是保留 bridge 网络
- 将运行必需文件复制到各自正确的主机路径，而不只是复制到 Quadlet 单元目录

## 运行模式

- `advice`：解释映射方式或审查输入，不写最终产物
- `design`：执行 planning 和 finalize 审阅，但在生成可运行产物前停止
- `generate`：在 planning 和 finalize 审阅通过后，生成已批准的可运行产物

## 文档说明

- `SKILL.md`：运行模式、工作流和高层规则
- `references/compose-mapping.md`：Compose 字段映射与拓扑决策
- `references/env-strategy.md`：env 处理、完整性校验与 typo 检测
- `references/github-repo-intake.md`：仓库发现与 canonical 输入选择
- `references/deployment-notes.md`：部署说明
- `references/validation.md`：验证与排障

## 限制

本 skill 不保证 Docker Compose 语义与 Podman Quadlet 语义完全等价。

## 许可证

MIT
