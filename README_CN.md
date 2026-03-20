# 飞书频道 — Claude Code 插件

通过 MCP Server 将飞书机器人接入 Claude Code。

机器人在飞书上接收消息，转发到你的 Claude Code 会话；Claude 通过 `reply` / `react` / `edit_message` 工具回复。

> 📖 [English Documentation](README.md)

## 前置要求

- [Bun](https://bun.sh) — 运行时环境。安装：
  `curl -fsSL https://bun.sh/install | bash`

## 快速上手

> 以下是单人 DM 场景的默认配对流程。群聊和多人设置请参考 [ACCESS.md](ACCESS.md)。

### 1. 创建飞书机器人

打开 [飞书开放平台](https://open.feishu.cn/app) → **创建自建应用**，填写名称（如 "ClaudeCode"）和描述。

#### 1a. 复制凭证

左侧菜单 → **凭证与基础信息**：
- 复制 **App ID**（`cli_xxx`）和 **App Secret**，第 3 步会用到。

![复制 App ID 和 App Secret](images/step1a-credentials.png)

#### 1b. 开启机器人能力

左侧菜单 → **应用能力** → **机器人**：
- 打开 **启用机器人** 开关。开启后用户才能在飞书中搜索到并 DM 这个机器人。

![开启机器人能力](images/step1b-bot-capability.png)

#### 1c. 配置权限

左侧菜单 → **权限管理** → **API 权限**：

| 权限 | 说明 | 是否必须 |
|---|---|---|
| `im:message` | 读取消息 | ✅ 必须 |
| `im:message:send_as_bot` | 以机器人身份发送消息 | ✅ 必须 |
| `im:resource` | 下载消息中的图片/文件 | ✅ 收发图片必须 |
| `im:message.reactions:write` | 添加表情回应 | 可选（`ackReaction` 功能需要） |
| `im:chat:readonly` | 读取会话信息 | 可选 |

搜索每个权限名称，点击 **开通**。

![权限配置页面](images/step1c-permissions.png)

#### 1d. 配置事件订阅 ⚠️ 关键步骤

> **这是最容易出错的地方。** 如果跳过此步或选错订阅方式，机器人能发消息但 **收不到消息**。

左侧菜单 → **事件与回调**：

1. **订阅方式** — 选择 **使用长连接接收事件**（WebSocket 模式，无需公网 URL）
2. 点击 **添加事件** → 搜索 `im.message.receive_v1` → 添加
   （完整名称：「接收消息 v2.0」）

![事件订阅：长连接 + im.message.receive_v1](images/step1d-event-subscription.png)

#### 1e. 发布版本

左侧菜单 → **版本管理与发布**：
- 点击 **创建版本** → 填写版本说明 → **提交发布**
- 组织的飞书管理员需要在 [管理后台](https://feishu.cn/admin/appCenter/audit) 审批。个人/测试租户可能自动通过。

> ⚠️ 每次修改权限或事件订阅后，必须**重新发布版本**才能生效。这一点很容易遗忘。

![发布新版本](images/step1e-publish.png)

#### 1f. 验证机器人上线

打开飞书 → 搜索机器人名称 → 应该能看到它作为联系人出现。如果找不到，检查版本是否已发布并审批通过。

### 2. 安装插件

以下是 Claude Code 命令，需要先启动 `claude` 会话。

首先注册插件市场（只需一次）：

```bash
claude plugin marketplace add V1ki/claude-feishu-plugin
```

然后安装插件：

```bash
claude plugin install feishu@claude-feishu-plugin
```

重启会话或执行 `/reload-plugins`。验证 `/feishu:configure` 能 tab 补全。

### 3. 保存凭证

```
/feishu:configure cli_xxx your_app_secret
```

将 `FEISHU_APP_ID=...` 和 `FEISHU_APP_SECRET=...` 写入
`~/.claude/channels/feishu/.env`。也可以手动编辑该文件，或通过环境变量设置（环境变量优先级更高）。

### 4. 带频道标志启动

退出当前会话，重新启动：

```bash
claude --dangerously-load-development-channels plugin:feishu@claude-feishu-plugin
```

> **说明：** `--channels` 需要插件在 Claude 的已批准频道白名单中，目前第三方插件暂不支持。
> `--dangerously-load-development-channels` 功能完全相同，只是跳过白名单检查。

### 5. 配对

在飞书中 DM 你的机器人，它会回复一个 6 位配对码。在 Claude Code 会话中：

```
/feishu:access pair <配对码>
```

之后你的消息就会直接到达 Claude。

### 6. 锁定策略

配对的目的是获取用户 ID。完成后切换到 `allowlist` 模式，避免陌生人收到配对码回复：

```
/feishu:access policy allowlist
```

## 访问控制

详见 **[ACCESS.md](ACCESS.md)**，包括 DM 策略、群聊支持、@提及检测、投递配置、skill 命令和 `access.json` 格式。

快速参考：ID 是飞书 open_id（如 `ou_xxx`）。默认策略 `pairing`。`ackReaction` 使用飞书表情类型（如 `THUMBSUP`）。

## 暴露给 Claude 的工具

| 工具 | 用途 |
| --- | --- |
| `reply` | 在飞书中发送消息。传入 `chat_id` + `text`，可选 `reply_to`（消息 ID）用于回复线程，`files`（绝对路径数组）用于附件。图片作为图片消息发送，其他类型作为文件。单个附件最大 50 MB。长文本自动分片。 |
| `react` | 给消息添加表情回应。使用飞书表情类型（`THUMBSUP`、`HEART`、`SMILE` 等）。 |
| `edit_message` | 编辑机器人之前发送的消息。适合「处理中…」→ 最终结果的进度更新。 |

## 图片支持

收到的图片自动下载到 `~/.claude/channels/feishu/inbox/`，本地路径通过 `<channel>` 通知传递给 Claude，可以用 `Read` 工具读取。

## Lark 国际版

在 `.env` 中设置：

```
FEISHU_API_BASE=https://open.larksuite.com/open-apis
```

## 工作原理

插件使用官方 [@larksuiteoapi/node-sdk](https://github.com/larksuite/node-sdk)
的 `WSClient` 建立 WebSocket 长连接：

- **无需公网 IP 或域名** — 本地开发环境即可运行
- **无需防火墙/白名单配置** — 只需能访问外网
- 认证在连接时完成，后续事件以明文传输

## 多 Claude Code 会话

一个飞书机器人 = 一条 WebSocket 连接接收事件。与 Telegram/Discord
不同（新连接自动踢掉旧连接），飞书允许多条连接共存但只将事件投递给**其中一个**。

本插件模拟 Telegram/Discord 的「后者优先」行为：

- **最新会话自动接管。** 新会话启动时向旧 server 进程发送 SIGTERM，接管
  WebSocket 连接。
- **旧会话保留发送能力。** 旧 server 进程优雅退出后，其 Claude Code 会话失去
  消息接收能力，但 `reply` / `react` / `edit_message` 仍可通过 REST API 使用。
- **自动清理。** Claude Code 退出时，server 检测到 stdin 关闭自动退出，释放锁。

如果需要多个完全独立的会话同时接收消息，需要创建多个飞书应用（各自拥有独立的 App ID / App Secret）。

## 常见问题排查

**机器人能发消息但收不到消息**

最常见原因：事件订阅未正确配置。
1. 飞书开放平台 → 你的应用 → **事件与回调**
2. 确认订阅方式为 **长连接**，不是 HTTP 回调
3. 确认已添加 `im.message.receive_v1` 事件
4. 确认修改后已**重新发布版本**
5. 检查残留进程：`ps aux | grep "bun.*server.ts"` — 杀掉旧进程

**收到配对码，但之后的消息不到达**

配对成功后需要重启 Claude Code（带 `--dangerously-load-development-channels` 参数）。

**`reply failed: chat oc_xxx is not allowlisted`**

这是配对后首条消息的已知问题，最新版已修复（`allowedChatIds` 运行时集合跟踪 `chat_id ↔ sender` 映射）。更新到最新插件版本即可。

**多个 Claude Code 会话，只有一个收到消息**

这是预期行为 — 参见 [多 Claude Code 会话](#多-claude-code-会话)。最新启动的会话始终接管。

## 卸载

```bash
claude plugin uninstall feishu
claude plugin marketplace remove claude-feishu-plugin
```
