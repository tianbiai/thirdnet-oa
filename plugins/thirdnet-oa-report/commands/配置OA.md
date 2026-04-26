---
name: 配置OA
description: 配置 OA 日报助手连接信息
allowed-tools:
  - AskUserQuestion
---

配置 OA 日报助手。

按照 `thirdnet-oa-report/skills/setup-oa-config/SKILL.md` 中定义的 OA 配置引导 Skill 工作流程执行。

关键步骤：
1. 调用 MCP 工具 `check_config` 检查当前配置状态
2. 使用 AskUserQuestion 一次性收集所有配置项（OA 地址、手机号、密码、SM2 公钥、HMAC 凭据）
3. 保存配置并调用 MCP 工具 `login` 测试连接
4. 调用 MCP 工具 `get_institutions` 获取机构列表供用户选择默认机构
5. 展示配置完成信息

安全约束：不在对话输出中显示密码原文。
