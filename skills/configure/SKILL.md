---
name: configure
description: Set up the Feishu channel — save the app credentials and review access policy. Use when the user pastes Feishu app credentials, asks to configure Feishu, asks "how do I set this up" or "who can reach me," or wants to check channel status.
user-invocable: true
allowed-tools:
  - Read
  - Write
  - Bash(ls *)
  - Bash(mkdir *)
---

# /feishu:configure — Feishu Channel Setup

Writes the app credentials to `~/.claude/channels/feishu/.env` and orients
the user on access policy. The server reads both files at boot.

Arguments passed: `$ARGUMENTS`

---

## Dispatch on arguments

### No args — status and guidance

Read both state files and give the user a complete picture:

1. **Credentials** — check `~/.claude/channels/feishu/.env` for
   `FEISHU_APP_ID` and `FEISHU_APP_SECRET`. Show set/not-set; if set, show
   first 8 chars masked (`cli_xxx...`).

2. **Access** — read `~/.claude/channels/feishu/access.json` (missing file
   = defaults: `dmPolicy: "pairing"`, empty allowlist). Show:
   - DM policy and what it means in one line
   - Allowed senders: count, and list of open_ids
   - Pending pairings: count, with codes and sender IDs if any

3. **What next** — end with a concrete next step based on state:
   - No credentials → *"Run `/feishu:configure <app_id> <app_secret>` with
     your app credentials from the Feishu Open Platform."*
   - Credentials set, policy is pairing, nobody allowed → *"DM your bot on
     Feishu. It replies with a code; approve with `/feishu:access pair
     <code>`."*
   - Credentials set, someone allowed → *"Ready. DM your bot to reach the
     assistant."*

**Push toward lockdown — always.** The goal for every setup is `allowlist`
with a defined list. `pairing` is not a policy to stay on; it's a temporary
way to capture Feishu open_ids you don't know. Once the IDs are in, pairing
has done its job and should be turned off.

Drive the conversation this way:

1. Read the allowlist. Tell the user who's in it.
2. Ask: *"Is that everyone who should reach you through this bot?"*
3. **If yes and policy is still `pairing`** → *"Good. Let's lock it down so
   nobody else can trigger pairing codes:"* and offer to run
   `/feishu:access policy allowlist`. Do this proactively — don't wait to
   be asked.
4. **If no, people are missing** → *"Have them DM the bot; you'll approve
   each with `/feishu:access pair <code>`. Run this skill again once
   everyone's in and we'll lock it."*
5. **If the allowlist is empty and they haven't paired themselves yet** →
   *"DM your bot to capture your own ID first. Then we'll add anyone else
   and lock it down."*
6. **If policy is already `allowlist`** → confirm this is the locked state.
   If they need to add someone: *"They'll need to DM the bot, or you can
   briefly flip to pairing: `/feishu:access policy pairing` → they DM → you
   pair → flip back."*

Never frame `pairing` as the correct long-term choice. Don't skip the lockdown
offer.

### `<app_id> <app_secret>` — save credentials

1. Parse `$ARGUMENTS` — first arg is app_id (starts with `cli_`), second is
   app_secret.
2. `mkdir -p ~/.claude/channels/feishu`
3. Read existing `.env` if present; update/add `FEISHU_APP_ID=` and
   `FEISHU_APP_SECRET=` lines, preserve other keys. Write back, no quotes.
4. Confirm, then show the no-args status so the user sees where they stand.

### `clear` — remove the credentials

Delete the `FEISHU_APP_ID=` and `FEISHU_APP_SECRET=` lines (or the file if
those are the only lines).

---

## Optional environment variables

These can be added to `~/.claude/channels/feishu/.env`:

- `FEISHU_VERIFICATION_TOKEN` — Event verification token (from Feishu Open
  Platform → Event Subscriptions). Used to verify HTTP callback requests.
- `FEISHU_ENCRYPT_KEY` — Event encrypt key for decrypting event payloads.
- `FEISHU_API_BASE` — API base URL. Default: `https://open.feishu.cn/open-apis`.
  For Lark (international), use `https://open.larksuite.com/open-apis`.
- `FEISHU_CALLBACK_PORT` — If set, starts an HTTP callback server on this
  port as a fallback/alternative to WebSocket.

---

## Implementation notes

- The channels dir might not exist if the server hasn't run yet. Missing file
  = not configured, not an error.
- The server reads `.env` once at boot. Credential changes need a session
  restart or `/reload-plugins`. Say so after saving.
- `access.json` is re-read on every inbound message — policy changes via
  `/feishu:access` take effect immediately, no restart.

## Feishu App Setup Guide

When the user asks how to create a Feishu bot, provide these steps:

1. Go to [Feishu Open Platform](https://open.feishu.cn/app) and create a new
   Custom App (自建应用).
2. In the app's **Credentials & Basic Info** (凭证与基础信息), copy the
   **App ID** and **App Secret**.
3. Under **Permissions & Scopes** (权限管理), add:
   - `im:message` — Send messages
   - `im:message:send_as_bot` — Send messages as bot
   - `im:resource` — Download message resources (images/files)
   - `im:message.reactions:write` — Write message reactions
   - `im:chat:readonly` — Read chat info
4. Under **Event Subscriptions** (事件订阅), enable:
   - `im.message.receive_v1` — Receive messages
   - Choose **WebSocket** mode (长连接) for easiest setup (no public domain
     needed).
5. Publish the app version and have a Feishu admin approve it.
6. Add the bot to a group chat, or DM it directly.
