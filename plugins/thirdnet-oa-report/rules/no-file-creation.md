---
name: no-file-creation
description: >
  禁止在用户项目目录下创建任何文件或运行包管理命令。所有 OA 操作必须通过 MCP 工具完成，
  不得绕过 MCP Server 用脚本、curl 或其他方式直接调用 API。
---

# 禁止在用户项目目录创建文件

OA 日报插件的所有操作必须通过 MCP Server 提供的工具完成，不允许任何替代方案。

## 绝对禁止

- 在用户项目目录下创建任何文件（.mjs、.js、.json、.sh、.py 等）
- 在用户项目目录下运行 `npm install`、`pip install` 等包管理命令
- 用 Bash 运行自定义脚本调用 OA API
- 用 `curl`、`fetch`、`wget` 等命令直接请求 OA 接口

## 正确做法

所有 OA 操作通过以下 MCP 工具完成：

| 操作 | MCP 工具 |
|------|---------|
| 检查配置 | `check_config` |
| 登录 | `login` |
| 获取模板 | `get_report_template` |
| 提交日报 | `submit_report` |
| 查询日报 | `get_reports` |
| 发送催办 | `send_reminder` |
| 获取机构 | `get_institutions` |

如果 MCP 工具不可用，应告知用户检查插件安装状态，而不是尝试绕过 MCP Server。
