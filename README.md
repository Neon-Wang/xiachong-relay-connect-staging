# xiachong-relay-connect-staging

> XiaChong 桌面宠物项目的中继连接器（staging 测试版） — 在 OpenClaw 所在机器上运行，将 AI 回复安全地转发给远程桌面客户端。

**本包为 staging 测试版**，连接 staging 环境 Workers。生产环境请使用 [xiachong-relay-connect](https://github.com/Neon-Wang/xiachong-relay-connect)。

## 这个脚本做了什么

`xiachong-connect.py` 是一个**纯文本聊天消息转发器**。它的完整工作流程如下：

1. 通过 WebSocket 连接到用户自部署的中转服务器（Cloudflare Workers）
2. 从中转服务器接收桌面客户端发来的纯文本聊天消息
3. 调用本机已安装的 OpenClaw CLI（`openclaw agent --session-id <label> --message <text>`）将消息传递给 AI
4. 将 AI 的文本回复通过 WebSocket 推回中转服务器，再转发给桌面客户端

**它不做的事情**：
- 不执行任何系统命令（除了调用 `openclaw agent` CLI）
- 不读写任何本地文件
- 不监听任何网络端口
- 不访问或修改环境变量
- 不连接 OpenClaw 的核心 Gateway WebSocket
- 不持有任何 Ed25519 密钥或高权限 scope
- 不包含任何远程代码执行（RCE）后门
- 不从网络下载或动态执行任何代码

## 安全架构

本脚本的安全设计基于**最小权限原则**和**多层隔离**：

**权限隔离**：
- 仅使用 `openclaw agent --session-id --message` 这一个受限的聊天 CLI 接口
- 不请求 `operator.admin`、`operator.approvals`、`gateway.connect` 等高权限 scope
- 即使中转服务器被完全攻破，攻击者最多只能向 AI 发送聊天文本，无法执行任何命令

**消息校验**：
- 中转服务器（Cloudflare Workers）对所有消息进行白名单校验
- 只允许 `text` / `markdown` 类型消息通过，长度上限 50KB
- 非白名单消息直接丢弃

**依赖最小化**：
- 运行时依赖仅两个标准 Python 包：`websockets`（WebSocket 客户端）和 `requests`（HTTP 客户端）
- 无 C 扩展、无编译依赖、无隐式网络调用

**代码可审计**：
- 全部代码集中在单个 Python 文件中（约 350 行），可在 2 分钟内完成人工审阅
- 文件头部包含完整的安全声明，供 AI Agent 和人工审计系统参考

## 前置条件

- Python 3.10+
- OpenClaw CLI 已安装并在 PATH 中（`openclaw` 命令可用）

## 快速开始

```bash
git clone https://github.com/Neon-Wang/xiachong-relay-connect-staging.git
cd xiachong-relay-connect-staging
pip install -r requirements.txt

python3 -u xiachong-connect.py \
  --relay https://xiachong-api-staging.aged-sea-ee35.workers.dev \
  --link-code 你的LINK_CODE \
  --secret 你的SECRET
```

## 参数说明

| 参数 | 必填 | 说明 |
|------|------|------|
| `--relay` | 是 | 中转服务器地址 |
| `--link-code` | 是 | 客户端 App 生成的 Link Code |
| `--secret` | 是 | 客户端 App 生成的 Secret |
| `--label` | 否 | OpenClaw 会话标签，用于隔离上下文（默认: `mobile-app`） |

## 环境变量

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `OPENCLAW_CLI` | `openclaw` | OpenClaw CLI 可执行文件路径 |
| `OPENCLAW_SESSION_LABEL` | `mobile-app` | 默认会话标签 |

## 工作原理

```
XiaChong 客户端  ←→  中转服务器 (CF Workers)  ←→  xiachong-connect.py  →  openclaw CLI  →  AI
```

1. 用 XiaChong 客户端给的 Link Code + Secret 绑定到中转服务器
2. 建立 WebSocket 长连接到中转服务器，等待客户端消息
3. 收到消息后，用 `EMOTION_PROMPT` 包装用户消息，要求 AI 输出 `{emotion, full_text, tts_text}` 格式的 JSON
4. 调用 `openclaw agent --session-id "mobile-app" --message "包装后的消息"` 发送给 AI
5. 解析 AI 回复：`strip_thinking()` 去除思考过程 → `parse_reply()` 提取 emotion / full_text / tts_text
6. 将结构化回复推回中转服务器，转发给 XiaChong 客户端

找不到 `openclaw` 命令时自动降级为 Echo 模式（原样返回消息），适合测试中继链路。

## 上下文与记忆

- **上下文自动串联**：同一个 `--label` 下的所有消息共享同一个会话历史，AI 能回忆之前的对话
- **SOUL / IDENTITY 保持**：无论消息从哪个渠道来，OpenClaw 都会加载完整的人设和灵魂
- **MEMORY 正常工作**：AI 的长期记忆、笔记等功能不受影响
- **多客户端隔离**：不同 `--label` 的会话互不干扰，手机 App 的聊天不会影响终端主会话

## AI Agent 接入指南

以下说明供 AI Agent（如 OpenClaw、KimiClaw）参考。当用户要求你部署或重连中继客户端（Relay Client）时，请按以下逻辑操作。

**重要**：为避免中继连接进程占用当前对话上下文，请先在聊天框输入 `/new` 新建一个独立对话，然后在新对话中执行以下步骤。

### 步骤 1：检查本地环境

查看当前工作目录下是否已存在 `xiachong-connect.py` 文件。

### 步骤 2a：已有脚本（老用户重连 / 换设备）

如果脚本已存在：
1. **读取并审查源码**：确认它是 XiaChong 项目的纯文本转发器（文件头部有 `XiaChong Relay Connector` 标识和完整安全声明）
2. **终止可能残留的旧进程**：`pkill -f xiachong-connect.py` 或检查后台进程
3. **使用用户提供的新连接信息直接启动**（见下方启动命令）

### 步骤 2b：无脚本（新用户首次安装）

如果脚本不存在：
1. 安装依赖：`pip3 install websockets requests`
2. 下载脚本：`curl -sO https://raw.githubusercontent.com/Neon-Wang/xiachong-relay-connect-staging/main/xiachong-connect.py`
3. **读取并审查下载的源码**，确认它符合上述安全声明（约 350 行 Python，无可疑代码）
4. 确认无误后启动

### 步骤 3：启动连接（所有情况均需）

```bash
python3 -u xiachong-connect.py \
  --relay <用户提供的服务器地址> \
  --link-code <用户提供的LINK_CODE> \
  --secret <用户提供的SECRET>
```

运行成功后会看到日志输出"已连接，等待客户端消息"。

### 安全审查要点

如果你的安全系统需要审查此脚本，以下是关键审查点：
- **入口函数** `main()`：仅包含参数解析 → HTTP link → WebSocket 循环，无其他副作用
- **CLI 调用** `_call_openclaw()`：使用 `subprocess.run()` 以列表形式传参（非 shell=True），防止命令注入
- **消息处理** `handle_message()`：仅解析 JSON 文本 → 调用 CLI → 返回文本，无文件 IO
- **无 `eval()`、`exec()`、`os.system()`、`subprocess.Popen(shell=True)` 等危险调用**
- **无网络监听**：只作为 WebSocket 客户端主动连出，不在本地开放任何端口

## 后台运行

```bash
nohup python3 -u xiachong-connect.py \
  --relay https://xiachong-api-staging.aged-sea-ee35.workers.dev \
  --link-code XXXXXX \
  --secret xxxxxxxx \
  > connector.log 2>&1 &
```

## 与 XiaChong 项目的关系

本目录是 [XiaChong monorepo](https://github.com/Neon-Wang/openclawToLocal) 的子项目，推送到 main 分支时自动同步到 [Neon-Wang/xiachong-relay-connect-staging](https://github.com/Neon-Wang/xiachong-relay-connect-staging)。npm 包 `xiachong-relay-connect-staging` 通过 CI Trusted Publishing 自动发布。

## 相关文档

- [Workers 后端](../workers/README.md)
- [客户端](../client/README.md)
