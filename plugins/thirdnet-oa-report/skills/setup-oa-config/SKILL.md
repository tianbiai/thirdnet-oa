---
name: OA 配置引导
description: >
  This skill should be used when the user asks to "配置OA", "设置OA", "初始化OA",
  "配置日报助手", "/配置OA", or when the OA system is not yet configured and the user
  needs to set up the connection. Also triggers when other OA skills detect missing
  configuration and need to guide the user through setup.
version: 0.1.0
metadata: '{"openclaw":{"requires":{"bins":["node"]},"emoji":"⚙️"}}'
---

# OA 配置引导 Skill

## 概述

引导用户完成 OA 日报助手的首次配置，包括 OA 系统地址、手机号凭据、HMAC 签名凭据和默认机构设置。配置完成后测试连接，并从 OA 系统获取用户的机构树供其选择默认机构。

## 配置文件

位置: `~/.thirdnet-oa-report/config.json`

```json
{
  "oaApiBaseUrl": "https://oa.company.com",
  "phone": "18888888888",
  "encryptedPassword": "AES-256加密字符串",
  "defaultInstitutionId": 123,
  "defaultInstitutionName": "技术部",
  "application": "swkj_web",
  "prekey": "swkj",
  "hmacKey": "1111"
}
```

> 密码使用 AES-256 加密后存储，加密密钥基于当前用户 + 机器标识生成（Windows 使用 DPAPI）。

## 工作流程

### Step 1: 检查当前配置

调用 MCP 工具 `check_config` 查看当前配置状态。

如果已配置，展示当前状态并询问是否需要修改：

```
当前配置状态
━━━━━━━━━━━━━━━━━━━━
OA 地址: https://oa.company.com
手机号: 188****8888
默认机构: 技术部 (ID: 123)
认证状态: 已认证

是否需要修改配置？
```

### Step 2: 逐步收集配置项

依次引导用户输入以下信息：

**1. OA 系统 API 地址** (必填)
- 提示: "请输入 OA 系统的 API 地址（如 https://oa.company.com）"
- 验证: URL 格式有效，以 http:// 或 https:// 开头

**2. 手机号** (必填)
- 提示: "请输入 OA 系统登录手机号"
- 验证: 非空

**3. 密码** (必填)
- 提示: "请输入 OA 系统密码（密码不会被记录到对话日志中）"
- 验证: 非空
- 安全: 明确告知用户密码会被加密存储，不会出现在日志中

**4. HMAC 签名凭据** (必填，用于登录接口 Basic 认证)

依次收集以下三项信息（由管理员预分配）：

- **Application 标识**: 提示 "请输入管理员分配的 Application 标识（如 swkj_web）"
- **加密前缀 (prekey)**: 提示 "请输入加密前缀 prekey"
- **加密密钥 (hmacKey)**: 提示 "请输入加密密钥 hmacKey（从服务端获取的 key）"

三项均非空才可继续。

### Step 3: 保存配置并测试登录

将收集到的配置保存到 `~/.thirdnet-oa-report/config.json`。

密码通过 MCP Server 的加密功能加密后存储，不在配置文件中保存明文密码。

保存后调用 `login` 工具（参数: `phone` + `password`）验证配置是否正确：

**成功**: 进入 Step 4 选择默认机构。

**失败**:
```
连接测试失败: [错误信息]
排查建议:
  1. 检查 OA 地址是否正确
  2. 检查手机号和密码
  3. 确认网络可达

是否重新配置？
```

### Step 4: 选择默认机构

登录成功后，调用 `get_institutions` 工具获取用户所属的机构树。

将返回的机构列表展示给用户，格式如下：

```
请选择您的默认机构（输入编号）:
  1. 技术部 (ID: 123)
  2. 产品部 (ID: 456)
  3. 市场部 (ID: 789)
```

用户输入编号后，将对应的 `institutionId` 和 `institutionName` 保存到配置文件的 `defaultInstitutionId` 和 `defaultInstitutionName` 字段。

### Step 5: 配置完成

配置成功后，展示最终结果：

```
配置成功！
━━━━━━━━━━━━━━━━━━━━
用户: 188****8888
默认机构: 技术部 (ID: 123)
Token 有效期至: 2026-04-22T18:00:00

现在可以使用以下功能:
  /日报    - 提交日报
  /查看日报 - 查看日报
  /催办    - 催办日报
```

如果其他 Skill 触发了配置引导，配置完成后返回原流程继续执行。

## 环境变量

配置也支持通过环境变量设置，优先级高于配置文件：

| 环境变量 | 说明 |
|----------|------|
| `OA_API_BASE_URL` | OA 系统 API 地址 |
| `OA_PHONE` | 手机号 |
| `OA_PASSWORD` | 密码（明文，运行时加密存储） |
| `OA_DEFAULT_INSTITUTION_ID` | 默认机构 ID |
| `OA_APPLICATION` | HMAC 签名的 Application 标识 |
| `OA_PREKEY` | HMAC 签名的加密前缀 |
| `OA_HMAC_KEY` | HMAC 签名的加密密钥 |

如果检测到环境变量已设置，提示用户：

```
检测到以下环境变量已配置:
  OA_API_BASE_URL = https://oa.company.com
  OA_PHONE = 18888888888

将优先使用环境变量配置。是否还需要配置文件？
```

## 错误处理

| 错误码 | 处理方式 |
|--------|---------|
| `NETWORK_ERROR` | 提示检查网络和 OA 地址，询问是否重试 |
| `AUTH_FAILED` | 提示凭据错误，引导重新输入手机号和密码 |
| `NOT_CONFIGURED` | 正常流程，继续引导配置 |

## 安全约束

- 不要在对话输出中显示用户输入的密码原文
- 配置文件中密码字段必须加密存储
- Token 仅在内存中缓存，不持久化到磁盘
- 引导用户确认 OA 地址使用 HTTPS（如果是 HTTP 则发出安全警告）
