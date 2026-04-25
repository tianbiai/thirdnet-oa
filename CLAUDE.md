# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

thirdnet-oa 是一个 Claude Code 插件项目，将企业 OA 系统的日报管理流程（提交、查看、催办、配置）集成到开发者日常工作环境中。项目通过 MCP Server 与 OA API 通信，利用 AI 自动总结工作内容生成日报。

## 项目结构

```
thirdnet-oa/
└── plugins/
    └── thirdnet-oa-report/       # OA 日报助手插件
        ├── .claude-plugin/plugin.json   # 插件元数据
        ├── .mcp.json                    # MCP Server 注册（node server/index.js）
        ├── package.json                 # Node.js 包配置
        ├── server/index.js              # 编译后的 MCP Server（1MB+，含所有依赖）
        ├── agents/oa-assistant.md       # 统一 OA 助手 Agent 定义
        ├── rules/confirm-before-submit.md  # 写操作确认规则
        └── skills/
            ├── submit-daily-report/SKILL.md   # 日报提交
            ├── view-daily-report/SKILL.md     # 日报查看
            ├── remind-daily-report/SKILL.md   # 日报催办
            └── setup-oa-config/SKILL.md       # OA 配置引导
```

## 架构

项目采用 Claude Code Plugin 架构：

- **MCP Server** (`server/index.js`)：编译打包的单文件 Node.js 服务，提供与 OA API 交互的所有 MCP 工具。包含 SM2 加密登录、HMAC-SHA512 Basic 认证、AES-256 密码加密等安全逻辑。
- **Agent** (`agents/oa-assistant.md`)：统一入口 Agent，处理所有 OA 日报相关工作流，定义 MCP 工具列表、输出格式规范、错误恢复策略和主动行为规则。
- **Skills**：四个独立工作流，分别处理配置、提交、查看、催办场景。每个 Skill 定义了详细的步骤流程和错误处理表。
- **Rules** (`rules/confirm-before-submit.md`)：全局写操作保护规则，强制所有提交/催办操作必须预览并等待用户确认。

## MCP 工具

Server 暴露以下工具供 Agent/Skill 调用：

| 工具 | 用途 |
|------|------|
| `check_config` | 检查 OA 配置和认证状态 |
| `login` | SM2 加密登录 |
| `get_report_template` | 获取日报模板字段定义 |
| `submit_report` | 提交日报（contents 为二维数组） |
| `get_reports` | 带筛选条件和分页的日报查询 |
| `send_reminder` | 向未提交用户发送催办（仅领导） |
| `get_institutions` | 获取机构树（含部门和成员） |

## 关键约束

- **写操作必须确认**：日报提交和催办发送前，必须通过 AskUserQuestion 展示预览内容并等待用户明确确认。OA 没有覆盖/撤回接口，提交后无法修改。
- **模板字段动态获取**：提交日报前必须先调用 `get_report_template` 查询实际字段定义，`field_key` 不可硬编码，不同部门/模板字段可能完全不同。
- **contents 格式**：`submit_report` 的 contents 参数为 `[[{field_key, field_value}]]` 二维数组结构。
- **日期转换**：催办 API 的 `period` 参数需将日期去连字符转为整数（如 `"2026-04-23"` → `20260423`）。
- **Token 管理**：Token 仅保存在内存中不持久化；`TOKEN_EXPIRED` 时自动重登录一次。
- **密码加密**：用户密码使用 AES-256 加密后存储于 `~/.thirdnet-oa-report/config.json`，密钥基于用户+机器标识生成（Windows DPAPI）。

## 配置文件

用户配置位于 `~/.thirdnet-oa-report/config.json`，包含 OA 地址、加密密码、默认机构 ID、HMAC 凭据。也支持通过 `OA_API_BASE_URL`、`OA_PHONE`、`OA_PASSWORD` 等环境变量覆盖。
