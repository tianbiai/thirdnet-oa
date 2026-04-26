---
name: setup-oa-config
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
// 示例值，实际使用时请替换为您的真实配置
{
  "oaApiBaseUrl": "https://your-oa-server.example.com/api",
  "phone": "13800138000",
  "encryptedPassword": "AES-256加密字符串",
  "defaultInstitutionId": 0,
  "defaultInstitutionName": "",
  "application": "your_application_name",
  "prekey": "your_prekey",
  "hmacKey": "your_hmac_key",
  "sm2PublicKey": "YOUR_BASE64_ENCODED_SM2_PUBLIC_KEY"
}
```

> 密码使用 AES-256 加密后存储，加密密钥基于当前用户 + 机器标识生成（Windows 使用 DPAPI）。

## 工作流程

### Step 1: 检查当前配置

调用 MCP 工具 `check_config` 查看当前配置状态。

**场景 A — 未配置**: 进入 Step 2 引导首次配置。

**场景 B — 已配置，用户选择修改**: 记录哪些字段发生了变更，进入 Step 2。修改配置后必须重新执行 Step 3 登录 → Step 4 查询机构树 → Step 5 更新部门信息，确保部门数据与新的配置一致。

**场景 C — 已配置，仅更新部门**: 如果用户只想切换默认部门而不改其他配置，跳过 Step 2，直接调用 `login` 刷新 Token，然后进入 Step 4 重新选择机构。

展示当前配置状态：

```
当前配置状态
━━━━━━━━━━━━━━━━━━━━
OA 地址: https://oa.company.com
手机号: 188****8888
默认机构: 技术部 (ID: 123)
认证状态: 已认证

请选择操作:
  1. 修改配置（修改后需重新选择部门）
  2. 仅切换默认部门
  3. 保持不变
```

### Step 2: 一次性收集全部配置项

使用 AskUserQuestion 工具，一次性向用户展示所有必填配置项，让用户确认或修改。展示时提供参考示例值帮助用户理解每个字段的含义。

**必须展示的字段及参考值：**

| 字段         | 说明                                 | 参考值                                     |
| ------------ | ------------------------------------ | ------------------------------------------ |
| oaApiBaseUrl | OA 系统 API 地址                     | `https://your-oa-server.example.com/api` |
| phone        | 登录手机号                           | `13800138000`                            |
| password     | 登录密码                             | （由用户填写）                             |
| publicKey    | SM2 加密公钥（Base64，用于登录加密） | `YOUR_BASE64_ENCODED_SM2_PUBLIC_KEY`     |
| application  | HMAC 签名的 Application 标识         | `your_application_name`                  |
| prekey       | HMAC 签名的加密前缀                  | `your_prekey`                            |
| hmacKey      | HMAC 签名的加密密钥                  | `your_hmac_key`                          |

**展示格式（使用 AskUserQuestion）：**

向用户展示一个包含所有字段的表单，每个字段附带参考示例值。用户可以：

- 直接确认使用示例值
- 修改任意字段为自己的实际值
- 选择 "Other" 自由输入

示例提示词：

```
请确认以下 OA 配置信息（如需修改请直接编辑对应字段）：

oaApiBaseUrl: https://your-oa-server.example.com/api
phone: （请输入您的手机号）
password: （请输入您的密码）
publicKey: （请输入 SM2 公钥）
application: （请输入 Application 标识）
prekey: （请输入加密前缀）
hmacKey: （请输入 HMAC 密钥）
```

**字段映射说明：**

- 用户提供的 `publicKey` 对应配置文件中的 `sm2PublicKey` 字段
- 密码不在对话输出中显示原文，配置文件中加密存储

**验证规则：**

- `oaApiBaseUrl`: 非空，以 http:// 或 https:// 开头（HTTP 地址需发出安全警告）
- `phone`: 非空
- `password`: 非空
- `publicKey`: 非空，Base64 格式
- `application`: 非空
- `prekey`: 非空
- `hmacKey`: 非空

所有字段验证通过后进入 Step 3。

### Step 3: 保存配置并测试登录

将收集到的配置保存到 `~/.thirdnet-oa-report/config.json`。

密码通过 MCP Server 的加密功能加密后存储，不在配置文件中保存明文密码。

保存后调用 `login` 工具（参数: `phone` + `password`）验证配置是否正确：

**成功**: 进入 Step 4 查询机构树并更新部门信息。

**失败**:

```
连接测试失败: [错误信息]
排查建议:
  1. 检查 OA 地址是否正确
  2. 检查手机号和密码
  3. 确认网络可达

是否重新配置？
```

### Step 4: 查询机构树并更新部门信息

登录成功后，**必须**调用 `get_institutions` 工具重新获取用户所属的机构树。即使配置未修改（场景 C），也应重新查询以获取最新的部门结构。

机构树返回的是嵌套结构，每个节点包含 `id`、`name`、`parent_id`、`children`。将机构树扁平化展示给用户，用缩进表示层级关系：

```
请选择您的默认机构（输入编号）:
  1. 技术部 (ID: 123)
    2. 前端组 (ID: 456) ← 技术部子部门
    3. 后端组 (ID: 789) ← 技术部子部门
  4. 产品部 (ID: 234)
  5. 市场部 (ID: 345)
```

用户输入编号后，将对应的 `institutionId` 和 `institutionName` 保存到配置文件的 `defaultInstitutionId` 和 `defaultInstitutionName` 字段。

**注意**: 机构树数据随组织架构变动而变化，每次配置修改或部门切换时都应重新查询，不要使用缓存的旧数据。

### Step 5: 配置完成

配置成功后，展示最终结果：

```
配置成功！
━━━━━━━━━━━━━━━━━━━━
用户: 188****8888
默认机构: 技术部 (ID: 123)
部门信息: 已更新（共 X 个机构）
Token 有效期至: 2026-04-22T18:00:00

现在可以使用以下功能:
  /日报    - 提交日报
  /查看日报 - 查看日报
  /催办    - 催办日报
```

如果其他 Skill 触发了配置引导，配置完成后返回原流程继续执行。

## 环境变量

配置也支持通过环境变量设置，优先级高于配置文件：

| 环境变量                      | 说明                         |
| ----------------------------- | ---------------------------- |
| `OA_API_BASE_URL`           | OA 系统 API 地址             |
| `OA_PHONE`                  | 手机号                       |
| `OA_PASSWORD`               | 密码（明文，运行时加密存储） |
| `OA_DEFAULT_INSTITUTION_ID` | 默认机构 ID                  |
| `OA_APPLICATION`            | HMAC 签名的 Application 标识 |
| `OA_PREKEY`                 | HMAC 签名的加密前缀          |
| `OA_HMAC_KEY`               | HMAC 签名的加密密钥          |
| `OA_SM2_PUBLIC_KEY`         | SM2 加密公钥（Base64 编码）  |

如果检测到环境变量已设置，提示用户：

```
检测到以下环境变量已配置:
  OA_API_BASE_URL = https://oa.company.com
  OA_PHONE = 18888888888

将优先使用环境变量配置。是否还需要配置文件？
```

## 错误处理

| 错误码             | 处理方式                               |
| ------------------ | -------------------------------------- |
| `NETWORK_ERROR`  | 提示检查网络和 OA 地址，询问是否重试   |
| `AUTH_FAILED`    | 提示凭据错误，引导重新输入手机号和密码 |
| `NOT_CONFIGURED` | 正常流程，继续引导配置                 |

## 安全约束

- 不要在对话输出中显示用户输入的密码原文
- 配置文件中密码字段必须加密存储
- Token 仅在内存中缓存，不持久化到磁盘
- 引导用户确认 OA 地址使用 HTTPS（如果是 HTTP 则发出安全警告）
