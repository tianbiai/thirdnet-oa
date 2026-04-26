---
name: submit-daily-report
description: >
  This skill should be used when the user asks to "提交日报", "写日报", "提交今天的日报",
  "帮我写日报", "日报", "/日报", "/submit-report", or wants to submit a daily work report
  to the OA system. Also triggers when user mentions daily report submission in Chinese
  (e.g., "我今天做了...", "帮我总结今天的工作").
version: 0.1.0
metadata: '{"openclaw":{"requires":{"bins":["node"]},"emoji":"📝"}}'
---
# OA 日报提交 Skill

## 概述

引导用户完成 OA 日报提交流程：检查配置 → 查询模板 → 收集/生成日报内容 → 构造数据 → 预览确认 → 提交到 OA 系统。支持手动输入和 AI 智能总结两种模式。

## 工作流程

### Step 1: 检查配置

调用 MCP 工具 `check_config` 检查 OA 连接状态。

- 如果 `configured` 为 `false`，提示用户先运行配置引导：引导用户使用 `/配置OA` 完成初始配置。
- 如果 `configured` 为 `true` 但 `authenticated` 为 `false`，调用 `login` 工具自动登录。
- 登录失败时，提示凭据错误并引导重新配置。

从返回结果中提取 `default_institution_id`，供后续步骤使用。

### Step 2: 查询模板

调用 MCP 工具 `get_report_template` 获取日报模板字段定义。

**参数**:

- `institution_id`: 从 Step 1 获取的 `default_institution_id`
- `date`: 用户指定日期或今天，格式 YYYY-MM-DD

**返回字段定义**（结构示意，实际字段由服务端决定）:

```json
{
  "template_id": 123,
  "fields": [
    { "field_key": "...", "field_name": "工作内容", "field_type": "String" }
  ]
}
```

**必须使用 Step 2 返回的实际 `field_key` 值**，不要假设或硬编码字段名。不同部门/模板的字段可能完全不同（例如 `gznr`、`gzjhw` 等）。记录每个字段的 `field_key` 和 `field_name`，供后续步骤使用。

- 如果返回 `TEMPLATE_NOT_FOUND`，提示用户联系管理员配置部门汇报模板，流程终止。

### Step 3: 收集日报内容

判断用户意图，选择以下模式之一：

**手动模式** — 用户直接提供了工作内容描述：

- 根据模板字段定义引导用户填写各字段内容
- 将用户描述按 `field_name` 分类对应到各 `field_key`
- 补充 date（默认今天，格式 YYYY-MM-DD）

**智能模式** — 用户请求 AI 帮助总结（如"帮我写日报"、"总结今天的工作"）：

- 分析当前对话上下文中的工作内容：
  - 代码变更记录（文件编辑、创建）
  - 技术讨论要点（问题诊断、方案决策）
  - 任务完成状态（已完成、进行中、阻塞项）
- 按模板字段定义将内容分类，生成各 `field_key` 对应的 `field_value`
- 如果对话上下文中没有足够信息，主动询问用户补充

### Step 4: 构造 contents 数据

将收集到的内容组装为 `contents` 二维数组格式。

**结构**（`field_key` 必须替换为 Step 2 查询到的实际值）:

```json
[
  [
    { "field_key": "实际查询到的key", "field_value": "1. 完成项目A的接口开发\n2. 修复登录页面样式" }
  ]
]
```

- 外层数组支持多组内容（如多条工作记录），内层数组为单组中的所有字段键值对
- `field_key` 必须与 Step 2 查询到的模板字段一致
- 通常情况只有一组内容（单个内层数组）

### Step 5: 预览并等待用户确认

根据模板的 `field_name` 展示生成的日报内容供用户审核，格式如下：

```
📋 日报预览
━━━━━━━━━━━━━━━━━━━━
📅 日期: 2026-04-23
🏢 部门: 技术部

【工作内容】
1. 完成项目A的接口开发
2. 修复登录页面样式问题

【下一步计划】
1. 进行项目A的联调测试
━━━━━━━━━━━━━━━━━━━━
```

- 每个字段以 `【field_name】` 作为标题
- 字段内容条目化展示（编号列表）

**必须使用 AskUserQuestion 工具询问用户是否确认提交**，提供以下选项：

- "确认提交" — 进入 Step 6 执行提交
- "需要修改" — 根据用户反馈更新内容，回到 Step 3 重新生成后再次预览
- "取消" — 终止流程

**这一步不可跳过。** 无论用户之前是否说过"直接提交"、"帮我提交"等话术，都必须展示预览并等待确认。原因是 OA 没有覆盖/撤回接口，提交后无法修改。

### Step 6: 提交日报

收到用户明确确认后，调用 MCP 工具 `submit_report` 提交日报。

**参数映射**:

- `institution_id`: 配置中的 `default_institution_id`（来自 Step 1）
- `date`: 用户指定日期或今天（YYYY-MM-DD）
- `contents`: Step 4 构造的二维数组

**处理返回结果**:

- 成功: 进入 Step 7 展示成功信息
- `DUPLICATE_REPORT`: 提示日报已存在。注意 OA API 没有覆盖/更新接口，无法重新提交。建议用户使用 `/查看日报` 查看已有汇报内容
- 其他错误: 展示错误信息，建议用户检查后重试

### Step 7: 反馈结果

展示最终结果：

```
✅ 日报提交成功！
报告编号: 123
提交日期: 2026-04-23
提交部门: 技术部
```

或错误信息：

```
❌ 提交失败: [错误信息]
💡 建议: [排查建议]
```

## 错误处理

| 错误码                 | 处理方式                                                               |
| ---------------------- | ---------------------------------------------------------------------- |
| `NOT_CONFIGURED`     | 引导用户运行 `/配置OA`                                               |
| `AUTH_FAILED`        | 提示检查凭据，引导重新配置                                             |
| `TOKEN_EXPIRED`      | 自动重登录后重试（最多 1 次）                                          |
| `TEMPLATE_NOT_FOUND` | 提示当前部门未配置汇报模板，建议联系管理员                             |
| `DUPLICATE_REPORT`   | 提示当日日报已存在，OA 无覆盖接口，建议使用 `/查看日报` 查看已有内容 |
| `NETWORK_ERROR`      | 提示检查网络连接和 OA 地址                                             |
| `PERMISSION_DENIED`  | 提示权限不足，联系管理员                                               |
| `OA_ERROR`           | 展示原始错误信息                                                       |

## 重要约束

- **日报提交前必须使用 AskUserQuestion 工具展示完整预览并等待用户确认，禁止跳过此步骤直接调用 submit_report**
- 不要在输出中显示用户的密码原文
- 智能总结时，如果对话上下文中没有足够信息，主动询问用户补充
- 日期格式统一使用 YYYY-MM-DD
- 提交前必须先查询模板，`field_key` 必须与模板字段一致
