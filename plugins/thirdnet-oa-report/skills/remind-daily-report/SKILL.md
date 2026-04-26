---
name: remind-daily-report
description: >
  This skill should be used when the user asks to "催办", "催日报", "催办日报",
  "提醒交日报", "看看谁没交日报", "/催办", "/remind", or wants to send reminders
  to team members who haven't submitted their daily reports. Also triggers when
  user wants to check who hasn't submitted and follow up.
version: 0.1.0
metadata: '{"openclaw":{"requires":{"bins":["node"]},"emoji":"📨"}}'
---
# OA 日报催办 Skill

## 概述

查询当日未提交日报的团队成员，确认后通过 OA 系统发送催办通知。催办由 OA 系统自动推送给机构内所有未提交人员，无需逐一指定。

## 工作流程

### Step 1: 检查配置

调用 `check_config` 确认 OA 已配置且已认证。未配置时引导至 `/配置OA`。

### Step 2: 查询提交情况与机构成员

同时调用以下两个工具，比对得出未提交名单：

**获取已提交日报** — 调用 `get_reports`：

- `start_time` = 今天（YYYY-MM-DD）
- `end_time` = 今天（YYYY-MM-DD）
- `institution_id` = 用户指定部门（可选，默认本部门）
- `period_type` = 1（日报）

如果用户指定了催办日期，使用用户指定的日期。

**获取机构全部成员** — 调用 `get_institutions`：

- 返回当前机构（institution）的完整成员列表，包含 `institution_id`。

**比对逻辑**：
从 `get_institutions` 返回的成员列表中，排除 `get_reports` 中已提交的成员，得到未提交人员名单。

### Step 3: 展示未提交名单

展示比对结果：

```
日报提交情况 (2026-04-22)
━━━━━━━━━━━━━━━━━━━━━━━━
已提交: 张三, 李四
未提交: 王五, 赵六, 孙七

未提交共 3 人
```

如果全部已提交，提示：

```
今日日报已全部提交！无需催办。
```

### Step 4: 确认催办

展示催办确认：

```
准备催办以下人员:
  1. 王五
  2. 赵六
  3. 孙七

催办将通过 OA 系统自动发送给以上未提交人员。
```

**必须使用 AskUserQuestion 工具询问用户是否确认发送催办**，提供以下选项：

- "确认发送" — 进入 Step 5 执行催办
- "取消" — 终止流程

**这一步不可跳过。** 催办是不可逆操作，发送后无法撤回。

### Step 5: 发送催办

收到用户明确确认后，调用 `send_reminder`：

- `institution_id`: 从 `get_institutions` 获取的机构 ID（number 类型）
- `period`: 催办日期转换后的整数值（number 类型）

**日期转换规则**：将日期字符串去掉连字符后转为整数。例如 `"2026-04-23"` 转为 `20260423`。

OA 系统会自动向该机构内所有未提交日报的人员发送催办通知，无需指定具体人员。

### Step 6: 反馈结果

`send_reminder` 返回纯文本字符串，直接展示给用户。

成功示例：

```
催办结果: 催促成功
```

部分成功示例：

```
催办结果: 催促成功 5/10。以下用户因未绑定邮箱未能催促：张三、李四
```

## 错误处理

| 错误码                | 处理方式                                 |
| --------------------- | ---------------------------------------- |
| `NOT_CONFIGURED`    | 引导 `/配置OA`                         |
| `AUTH_FAILED`       | 自动重登录后重试                         |
| `TOKEN_EXPIRED`     | 自动重登录后重试                         |
| `PERMISSION_DENIED` | 提示无催办权限，仅部门负责人可使用此功能 |
| `NETWORK_ERROR`     | 提示检查网络                             |
| `OA_ERROR`          | 展示原始错误信息                         |

## 重要约束

- **催办发送前必须使用 AskUserQuestion 工具展示催办名单并等待用户确认，禁止跳过此步骤直接调用 send_reminder**
- 展示催办对象时使用姓名而非 ID
- 仅部门负责人（leader）拥有催办权限，非负责人调用将返回 `PERMISSION_DENIED`
- OA 系统自动决定催办对象，Skill 层仅展示信息供确认
