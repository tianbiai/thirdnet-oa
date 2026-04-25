---
name: OA 日报查看
description: >
  This skill should be used when the user asks to "查看日报", "看日报", "今天的日报",
  "查看日报提交情况", "/查看日报", "/view-report", or wants to query and view daily
  reports from the OA system. Triggers on queries about report status, team submissions,
  or specific people's reports (e.g., "张三交了日报吗", "技术部本周日报情况").
version: 0.1.0
metadata: '{"openclaw":{"requires":{"bins":["node"]},"emoji":"📋"}}'
---

# OA 日报查看 Skill

## 概述

查询并展示 OA 系统中的日报数据。支持按日期、人员、部门等维度查询，以清晰格式化方式展示结果。

## 工作流程

### Step 1: 检查配置

调用 `check_config` 确认 OA 已配置且已认证。未配置时引导至 `/配置OA`。

### Step 2: 解析查询条件

从用户自然语言中提取查询参数：

**日期范围**（映射为 `start_time` / `end_time`）:
- "今天" → start_time = end_time = 今天
- "昨天" → start_time = end_time = 昨天
- "本周" → start_time = 本周一, end_time = 今天
- "最近三天" → start_time = 3天前, end_time = 今天
- 未指定 → 默认今天

**周期类型**:
- "日报" / 未指定 → period_type = 1
- "周报" → period_type = 2
- 可同时传入 `period` 指定具体周期值（如 20260423）

**目标人员/部门**:
- "我的" / 未指定 → user_id 为空（查自己）
- 指定人名 → 先调用 `get_institutions` 获取完整机构树，在 users 列表中查找匹配的 user_id
- 指定部门名称 → 先调用 `get_institutions` 查找匹配的机构节点，获取 institution_id
- 注意：`get_institutions` 返回的是完整机构树（含子部门和用户列表），应缓存该结果避免重复调用

### Step 3: 查询日报

调用 MCP 工具 `get_reports`，传入解析后的参数：
- `institution_id`: 部门机构 ID（通过 get_institutions 查得）
- `user_id`: 用户 ID（通过 get_institutions 查得）
- `period_type`: 周期类型（1=日报, 2=周报）
- `period`: 周期值（如 20260423）
- `start_time` / `end_time`: 日期范围（格式 YYYY-MM-DD）
- `index` / `size`: 分页参数（按需使用）

### Step 4: 格式化展示

根据查询场景选择合适的展示格式：

#### 个人日报详情视图

适用于：查看某人某天的日报详情。解析 `content_json`（JSON 字符串），按每个字段的 `field_name` 用【】括号展示标题，多值字段自动拆分为编号列表。

```
━━━ 📋 日报详情 ━━━━━━━━━━━━━━━━━━━━━━━━
  姓名：张三
  部门：技术部
  日期：2026-04-23 (周四)

  【工作内容】
  1. 完成项目A的用户模块接口开发
  2. 修复了登录页面的样式问题

  【下一步计划】
  1. 进行用户模块接口联调测试
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

content_json 解析规则：
1. 将 `content_json` 字符串解析为 JSON（通常是二维数组，每个元素包含 `field_key` 和 `field_value`）
2. 如有模板字段定义（field_name），用 field_name 作为标题；否则用 field_key
3. 单条内容直接显示，多条内容（含换行符或数组）自动拆分为编号列表

#### 团队日报总览视图

适用于：查看某部门/团队的日报提交情况。调用 `get_institutions` 获取该机构下所有成员列表，再与 `get_reports` 返回结果对比，区分已提交和未提交人员。

```
━━━ 📊 技术部 · 日报提交情况 ━━━━━━━━━━━━
  日期：2026-04-23 (周四)

  ✅ 已提交 (3人)
  ├─ 张三  ┃  完成3项工作，计划2项
  ├─ 李四  ┃  完成2项工作，计划1项
  └─ 王五  ┃  完成1项工作，计划2项

  ❌ 未提交 (2人)
  ├─ 赵六
  └─ 钱七

  提交率：3/5 (60%)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

已提交人员摘要生成规则：
1. 从 content_json 中解析各字段内容
2. 统计工作内容条数和计划条数，生成简要摘要（如 "完成3项工作，计划2项"）
3. 具体字段名称应根据实际模板动态适配

未提交人员判定规则：
1. 从 `get_institutions` 获取目标机构的所有 users
2. 从 `get_reports` 获取已提交的 user_id 列表
3. 差集即为未提交人员

#### 多日汇总视图

适用于：查看某人连续多天的日报提交情况。按天分组展示，未提交的天数标记为"未提交"，末尾统计提交天数。

```
━━━ 📅 张三 · 本周日报汇总 ━━━━━━━━━━━━━━

  ▌4/21 (周一)
    工作：完成用户模块接口开发、修复登录样式
    计划：进行联调测试

  ▌4/22 (周二) — 未提交

  本周提交：1/2 天
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

每日摘要生成规则：
1. 对已提交的日期，从 content_json 提取核心内容，用简洁的一句话概括各字段
2. 多条内容用顿号连接，控制在合理长度内
3. 未提交的日期显示"未提交"标记

### Step 5: 交互式深入

展示概览后，询问用户是否需要查看具体日报详情。如果用户指定查看某人日报，调用 `get_reports` 获取详细内容并展示个人日报详情视图。

## 查询示例

| 用户输入 | 解析参数 |
|---------|---------|
| "查看我今天的日报" | start_time/end_time=今天, user=自己, period_type=1 |
| "张三昨天提交了什么" | start_time/end_time=昨天, user=张三(查user_id), period_type=1 |
| "技术部本周的日报提交情况" | start_time=本周一/end_time=今天, institution=技术部(查institution_id), period_type=1 |
| "看看大家交了没" | start_time/end_time=今天, institution=本部门, period_type=1 |
| "查看本周周报" | start_time=本周一/end_time=今天, user=自己, period_type=2 |

## 错误处理

| 错误码 | 处理方式 |
|--------|---------|
| `NOT_CONFIGURED` | 引导 `/配置OA` |
| `AUTH_FAILED` | 自动重登录后重试 |
| `TOKEN_EXPIRED` | 自动重登录后重试 |
| `NETWORK_ERROR` | 提示检查网络 |
| 查无此人 | 提示人员名称可能不正确，建议通过 get_institutions 查看团队成员列表 |
| 查无此部门 | 提示部门名称可能不正确，建议通过 get_institutions 查看可用部门列表 |
