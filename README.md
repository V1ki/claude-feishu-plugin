# Feishu Channel for Claude Code

Connect a Feishu (飞书) bot to your Claude Code with an MCP server.

The MCP server logs into Feishu as a bot and provides tools to Claude to reply,
react, or edit messages. When you message the bot, the server forwards the
message to your Claude Code session.

## Prerequisites

- [Bun](https://bun.sh) — the MCP server runs on Bun. Install with
  `curl -fsSL https://bun.sh/install | bash`.

## Quick Setup

> Default pairing flow for a single-user DM bot. See [ACCESS.md](ACCESS.md)
> for groups and multi-user setups.

**1. Create a Feishu bot.**

Go to [Feishu Open Platform](https://open.feishu.cn/app) and create a new
Custom App (自建应用):

1. In **Credentials & Basic Info** (凭证与基础信息), copy the **App ID** and
   **App Secret**.
2. Under **Permissions & Scopes** (权限管理), add these scopes:
   - `im:message` — Send messages
   - `im:message:send_as_bot` — Send messages as bot
   - `im:resource` — Download message resources (images/files)
   - `im:message.reactions:write` — Write message reactions
   - `im:chat:readonly` — Read chat info
3. Under **Event Subscriptions** (事件订阅):
   - Enable `im.message.receive_v1` (Receive messages)
   - Choose **WebSocket** mode (长连接) — no public domain needed
4. Publish the app version and have a Feishu admin approve it.

**2. Add the marketplace and install the plugin.**

These are Claude Code commands — run `claude` to start a session first.

First, register this repository as a plugin marketplace (only needed once):

```bash
claude plugin marketplace add V1ki/claude-feishu-plugin
```

Then install the plugin:

```bash
claude plugin install feishu@claude-feishu-plugin
```

Restart your session or run `/reload-plugins`. Check that `/feishu:configure`
tab-completes.

**3. Give the server the credentials.**

```
/feishu:configure cli_xxx your_app_secret
```

Writes `FEISHU_APP_ID=...` and `FEISHU_APP_SECRET=...` to
`~/.claude/channels/feishu/.env`. You can also write that file by hand, or set
the variables in your shell environment — shell takes precedence.

**4. Relaunch with the channel flag.**

The server won't connect without this — exit your session and start a new one:

```bash
claude --dangerously-load-development-channels plugin:feishu@claude-feishu-plugin
```

> **Note:** `--channels` requires the plugin to be on Claude's approved
> channels allowlist, which is not yet available for third-party plugins.
> Use `--dangerously-load-development-channels` instead — it has the same
> functionality but skips the allowlist check.

**5. Pair.**

DM your bot on Feishu — it replies with a 6-character pairing code. In your
assistant session:

```
/feishu:access pair <code>
```

Your next DM reaches the assistant.

**6. Lock it down.**

Pairing is for capturing IDs. Once you're in, switch to `allowlist` so
strangers don't get pairing-code replies. Ask Claude to do it, or
`/feishu:access policy allowlist` directly.

## Access control

See **[ACCESS.md](ACCESS.md)** for DM policies, groups, mention detection,
delivery config, skill commands, and the `access.json` schema.

Quick reference: IDs are **Feishu open_ids** (e.g. `ou_xxx`). Default policy
is `pairing`. `ackReaction` uses Feishu emoji types like `THUMBSUP`.

## Tools exposed to the assistant

| Tool | Purpose |
| --- | --- |
| `reply` | Send to a chat. Takes `chat_id` + `text`, optionally `reply_to` (message ID) for native threading and `files` (absolute paths) for attachments. Images send as photos; other types send as files. Max 50MB each. Auto-chunks text; files send as separate messages after the text. Returns the sent message ID(s). |
| `react` | Add an emoji reaction to a message by ID. Uses Feishu emoji types (e.g. `THUMBSUP`, `HEART`, `SMILE`). |
| `edit_message` | Edit a message the bot previously sent. Useful for "working…" → result progress updates. Only works on the bot's own messages. |

## Photos

Inbound photos are downloaded to `~/.claude/channels/feishu/inbox/` and the
local path is included in the `<channel>` notification so the assistant can
`Read` it.

## Lark (International) support

For the international version (Lark), set in your `.env`:

```
FEISHU_API_BASE=https://open.larksuite.com/open-apis
```

## How it works

The plugin uses the official
[@larksuiteoapi/node-sdk](https://github.com/larksuite/node-sdk) `WSClient`
to establish a WebSocket long connection with the Feishu Open Platform. This
means:

- **No public IP or domain needed** — works in local dev environments
- **No firewall / whitelist config** — just outbound internet access
- Authentication happens at connection time; subsequent events are plaintext
- If multiple instances of the same app connect, only one receives each event

## Uninstall

```bash
claude plugin uninstall feishu
claude plugin marketplace remove claude-feishu-plugin
```
