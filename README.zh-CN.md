# Quadlet Migrator

[English](./README.md) | [简体中文](./README.zh-CN.md)

一个 skill，用来把 `docker run` 命令和 Docker Compose 配置转换成可审阅、可调整、可应用的 Podman Quadlet 文件。

## 功能概览

- 将 `docker run` 命令和 Docker Compose 配置转换成 Podman Quadlet 文件
- 默认先把生成文件写到当前目录，方便你在安装前先审阅
- 只有在你已经指定其他位置，或现有文件会发生冲突时，才询问输出位置
- 帮助在 `.container`、`.pod`、`.network`、`.volume`、`.build` 之间做选择；对有关联的多容器服务，通常优先用 pod
- 在合适时保留 `.env` / `env_file` 工作流
- 把庞杂的 env 模板收敛成用户真正需要确认的少量问题
- 可生成辅助脚本，例如 `install.sh`、`uninstall.sh`、`reload.sh`、`start.sh`、`stop.sh`、`restart.sh`
- 识别服务运行时仍需要的项目内文件，例如挂载配置、初始化数据和辅助脚本
- 在声称结果可运行前，检查 env 文件是否完整
- 在 planning 阶段先确认关键部署决策，并在审阅和执行阶段使用清晰的检查清单
- 当所选镜像使用包含仓库地址的完整镜像名时，例如 `docker.io/...` 或 `ghcr.io/...`，可选规划 `AutoUpdate=registry`
- 说明 rootless / rootful 安装路径、部署说明和验证步骤

## 设计原则

- 用能满足需求的最简单模式
- 将 planning、审阅、生成文件分成独立步骤
- 不虚构部署相关取值
- 如果映射会带来行为变化，要明确说出来
- 优先产出容易理解、容易维护的结果
- 先把文件写到当前目录审阅，再决定是否安装
- 对多容器服务，如果用 pod 更清晰，就优先用 pod
- 运行仍需要的额外文件保留在已审阅的输出中，并通过主机上的绝对路径引用，而不是复制进 Quadlet 单元目录

## 运行模式

- `advice`：解释映射方式、审查输入，或回答定向问题，不写最终文件
- `design`：执行 planning 和最后一轮交互式审阅，但在生成可运行文件前停止
- `generate`：执行 planning、最后一轮交互式审阅和 execution，然后生成已批准的可运行文件

## 工作流

工作流分为三个阶段：`Planning`、`Finalize`、`Execution`。

- `advice` 通常停留在 `Planning`，或直接回答一个聚焦的问题
- `design` 包含 `Planning` 和 `Finalize`
- `generate` 包含全部三个阶段

Planning 用来收集并确认仍未决定的部署问题。
Finalize 是在这些问题讨论清楚之后进行的对话式审阅。
只有在用户批准这次审阅后，才进入 Execution。

## 文档说明

- `SKILL.md`：运行模式、工作流和高层规则
- `references/compose-mapping.md`：Compose 字段映射与拓扑决策
- `references/env-strategy.md`：env 处理、完整性校验与 typo 检测
- `references/github-repo-intake.md`：说明 skill 如何找到正确的仓库入口
- `references/deployment-notes.md`：部署说明
- `references/validation.md`：验证与排障

## 限制

本 skill 不保证 Docker Compose 语义与 Podman Quadlet 语义完全等价。

## 许可证

MIT
