# Feishu — Access & Delivery

A Feishu bot is accessible to anyone within your organization (or anyone who
finds it). Without a gate, those messages would flow straight into your
assistant session. The access model described here decides who gets through.

By default, a DM from an unknown sender triggers **pairing**: the bot replies
with a 6-character code and drops the message. You run `/feishu:access pair
<code>` from your assistant session to approve them. Once approved, their
messages pass through.

All state lives in `~/.claude/channels/feishu/access.json`. The
`/feishu:access` skill commands edit this file; the server re-reads it on every
inbound message, so changes take effect without a restart. Set
`FEISHU_ACCESS_MODE=static` to pin config to what was on disk at boot (pairing
is unavailable in static mode since it requires runtime writes).

## At a glance

| | |
| --- | --- |
| Default policy | `pairing` |
| Sender ID | Feishu open_id (e.g. `ou_xxx`) |
| Group key | Feishu chat_id (e.g. `oc_xxx`) |
| `ackReaction` quirk | Uses Feishu emoji types like `THUMBSUP`, `HEART`, etc. |
| Config file | `~/.claude/channels/feishu/access.json` |

## DM policies

`dmPolicy` controls how DMs from senders not on the allowlist are handled.

| Policy | Behavior |
| --- | --- |
| `pairing` (default) | Reply with a pairing code, drop the message. Approve with `/feishu:access pair <code>`. |
| `allowlist` | Drop silently. No reply. Useful once all your users are paired. |
| `disabled` | Drop everything, including allowlisted users and groups. |

```
/feishu:access policy allowlist
```

## User IDs

Feishu identifies users by **open_ids** like `ou_xxx`. These are stable per
app and can be found in the Feishu Open Platform admin console. Pairing
captures the ID automatically.

```
/feishu:access allow ou_xxx
/feishu:access remove ou_xxx
```

## Groups

Groups are off by default. Opt each one in individually.

```
/feishu:access group add oc_xxx
```

Feishu group chat IDs look like `oc_xxx`. You can find them by adding the bot
to a group and checking the event payload, or from the Feishu admin console.

With the default `requireMention: true`, the bot responds only when @mentioned
or replied to. Pass `--no-mention` to process every message, or `--allow
id1,id2` to restrict which members can trigger it.

```
/feishu:access group add oc_xxx --no-mention
/feishu:access group add oc_xxx --allow ou_xxx1,ou_xxx2
/feishu:access group rm oc_xxx
```

## Mention detection

In groups with `requireMention: true`, any of the following triggers the bot:

- A structured @mention in the message
- A reply to one of the bot's messages
- A match against any regex in `mentionPatterns`

```
/feishu:access set mentionPatterns '["^hey claude\\b", "\\bassistant\\b"]'
```

## Delivery

Configure outbound behavior with `/feishu:access set <key> <value>`.

**`ackReaction`** reacts to inbound messages on receipt. Feishu uses emoji type
strings. Common types:

> THUMBSUP, THUMBSDOWN, HEART, FIRE, SMILE, CLAP, THINKING, SURPRISED,
> CRY, PARTY, COOL, OK, PRAY, MUSCLE, JIAYI, FINGERHEART

```
/feishu:access set ackReaction THUMBSUP
/feishu:access set ackReaction ""
```

**`replyToMode`** controls threading on chunked replies. When a long response
is split, `first` (default) threads only the first chunk under the inbound
message; `all` threads every chunk; `off` sends all chunks standalone.

**`textChunkLimit`** sets the split threshold. Default 4096 characters.

**`chunkMode`** chooses the split strategy: `length` cuts exactly at the limit;
`newline` prefers paragraph boundaries.

## Skill reference

| Command | Effect |
| --- | --- |
| `/feishu:access` | Print current state: policy, allowlist, pending pairings, enabled groups. |
| `/feishu:access pair a4f91c` | Approve pairing code `a4f91c`. Adds the sender to `allowFrom` and sends a confirmation on Feishu. |
| `/feishu:access deny a4f91c` | Discard a pending code. The sender is not notified. |
| `/feishu:access allow ou_xxx` | Add a user open_id directly. |
| `/feishu:access remove ou_xxx` | Remove from the allowlist. |
| `/feishu:access policy allowlist` | Set `dmPolicy`. Values: `pairing`, `allowlist`, `disabled`. |
| `/feishu:access group add oc_xxx` | Enable a group. Flags: `--no-mention`, `--allow id1,id2`. |
| `/feishu:access group rm oc_xxx` | Disable a group. |
| `/feishu:access set ackReaction THUMBSUP` | Set a config key: `ackReaction`, `replyToMode`, `textChunkLimit`, `chunkMode`, `mentionPatterns`. |

## Config file

`~/.claude/channels/feishu/access.json`. Absent file is equivalent to `pairing`
policy with empty lists, so the first DM triggers pairing.

```jsonc
{
  // Handling for DMs from senders not in allowFrom.
  "dmPolicy": "pairing",

  // Feishu open_ids allowed to DM.
  "allowFrom": ["ou_xxx"],

  // Groups the bot is active in. Empty object = DM-only.
  "groups": {
    "oc_xxx": {
      // true: respond only to @mentions and replies.
      "requireMention": true,
      // Restrict triggers to these senders. Empty = any member (subject to requireMention).
      "allowFrom": []
    }
  },

  // Case-insensitive regexes that count as a mention.
  "mentionPatterns": ["^hey claude\\b"],

  // Feishu emoji type string. Empty string disables.
  "ackReaction": "THUMBSUP",

  // Threading on chunked replies: first | all | off
  "replyToMode": "first",

  // Split threshold for long messages.
  "textChunkLimit": 4096,

  // length = cut at limit. newline = prefer paragraph boundaries.
  "chunkMode": "newline"
}
```
