# Thirdnet OA 日报助手

将企业 OA 系统的日报管理流程集成到 Claude Code 开发环境中，通过 AI 自动总结工作内容，让日报提交变得简单高效。

## 功能

| 功能 | 说明 | 触发方式 |
|------|------|----------|
| 日报提交 | AI 智能总结或手动填写工作内容，一键提交到 OA | `/日报` |
| 日报查看 | 按日期、人员、部门查询日报，支持个人详情和团队总览 | `/查看日报` |
| 日报催办 | 查询未提交名单，一键发送催办通知（需负责人权限） | `/催办` |
| OA 配置 | 引导完成首次配置，管理 OA 地址、凭据和默认部门 | `/配置OA` |

## 安装

### 前置要求

- [Claude Code](https://claude.ai/code) CLI 或桌面版
- Node.js（MCP Server 运行需要）

### 通过 Git 安装

```bash
# 克隆仓库
git clone https://github.com/tianbiai/thirdnet-oa.git
cd thirdnet-oa

# 在 Claude Code 中添加插件
claude plugin add .
```

### 手动安装

将仓库目录添加到 Claude Code 的插件路径中：

```bash
claude plugin add /path/to/thirdnet-oa
```

## 使用

### 1. 首次配置

安装后先运行配置向导，输入 OA 系统的连接信息和凭据：

```
> /配置OA
```

需要提供以下信息：

| 配置项 | 说明 |
|--------|------|
| OA 地址 | OA 系统 API 地址（如 `https://oa.company.com/api`） |
| 手机号 | 登录手机号 |
| 密码 | 登录密码（AES-256 加密存储） |
| SM2 公钥 | 用于登录加密的 SM2 公钥（Base64 编码） |
| Application | HMAC 签名标识 |
| Prekey | HMAC 签名前缀 |
| HMAC Key | HMAC 签名密钥 |

配置完成后会自动登录并获取机构树，选择你的默认部门即可。

### 2. 提交日报

```
> /日报
```

支持两种模式：

- **AI 智能总结**：根据当前对话上下文自动生成日报内容
- **手动填写**：按模板字段逐项填写

提交前会展示完整预览，确认后才会发送到 OA 系统。

### 3. 查看日报

```
> /查看日报
```

支持自然语言查询：

| 示例 | 含义 |
|------|------|
| "查看我今天的日报" | 查看个人日报详情 |
| "技术部本周的日报提交情况" | 查看团队提交总览 |
| "张三昨天提交了什么" | 查看指定人员日报 |
| "看看大家交了没" | 查看今日提交情况 |

### 4. 催办日报

```
> /催办
```

查看未提交人员名单，确认后发送催办通知。仅部门负责人有此权限。

## 配置文件

配置保存在 `~/.thirdnet-oa-report/config.json`，密码经 AES-256 加密后存储。

也支持通过环境变量配置（优先级高于配置文件）：

```bash
export OA_API_BASE_URL="https://oa.company.com/api"
export OA_PHONE="13800138000"
export OA_PASSWORD="your_password"
export OA_APPLICATION="your_app"
export OA_PREKEY="your_prekey"
export OA_HMAC_KEY="your_hmac_key"
export OA_SM2_PUBLIC_KEY="your_sm2_public_key"
export OA_DEFAULT_INSTITUTION_ID="123"
```

## 安全

- 密码使用 AES-256 加密存储，加密密钥基于用户+机器标识生成（Windows DPAPI）
- Token 仅在内存中缓存，不持久化到磁盘
- 登录通信使用 SM2 加密
- API 请求使用 HMAC-SHA512 签名认证
- 日报提交和催办等写操作必须用户确认后才执行

## 项目结构

```
thirdnet-oa/
└── plugins/
    └── thirdnet-oa-report/
        ├── .claude-plugin/plugin.json   # 插件元数据
        ├── .mcp.json                    # MCP Server 注册
        ├── server/index.js              # MCP Server（含所有依赖）
        ├── rules/                       # 安全规则
        │   ├── confirm-before-submit.md
        │   └── no-file-creation.md
        └── skills/                      # 技能定义
            ├── submit-daily-report/
            ├── view-daily-report/
            ├── remind-daily-report/
            └── setup-oa-config/
```

## License

MIT
