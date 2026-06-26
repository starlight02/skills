---
name: alma-channel-protocol
description: >-
  Complete reference for Alma's WebSocket channel bridge protocol — how built-in bridges
  (Telegram, Discord, Feishu/Lark, Weixin) send messages to the Alma server, how SENDER
  PROFILE matching works, how reply/quoting is handled, and how to build custom bridges
  such as QQ/OneBot or Slack. Use this skill whenever the user wants to: build a custom
  channel bridge for Alma, debug an Alma channel integration, understand `generate_response`
  over `ws://localhost:23001/ws/threads`, implement a QQ/OneBot bridge for Alma, design
  per-message `[From: ...]` / `[msg:N]` formatting, inject SENDER PROFILE blocks into
  `ephemeralContext`, implement reply/quoting (incoming `buildReplyContext` and outgoing
  reply references), inspect or write `channel_mappings`, choose the right `source` value
  for a custom platform, or anything involving alma-channel-protocol. Also trigger when the
  user mentions "Alma bridge", "Alma WebSocket", "Alma channel", "generate_response",
  "channel_mappings", "SENDER PROFILE", "ephemeralContext", "buildReplyContext",
  "message_delta", "message_updated", or asks why their custom Alma bridge sees empty
  assistant text. Prefer this skill before answering any question about how Alma talks to
  channels, even when the user does not name it directly.
---

# Alma Channel Bridge Protocol Reference

This skill is the authoritative protocol reference for Alma's channel bridges. It documents how built-in bridges talk to the Alma server over WebSocket and how to replicate that protocol for custom bridges (QQ/OneBot, Slack, Matrix, etc.). When a task is about *implementing* an Alma → IM-platform integration, this is the contract.

## Architecture

```
[Platform] → [Bridge Class] → [WebSocket] → [Alma Server]
 (Telegram,    (MessageBridge,   ws://         (generate_response,
  Discord,      DiscordBridge,    127.0.0.1     ephemeralContext assembly,
  Feishu,       FeishuBridge,     :23001        SENDER PROFILE stripping,
  Weixin,       WeixinBridge,     /ws/threads   memory extraction)
  QQ/OneBot)    CustomBridge)
```

The single architectural insight to internalize: **profile matching (SENDER PROFILE injection) happens in the bridge, NOT the server.** Each bridge scans `~/.config/alma/people/*.md` for matching platform IDs and injects the result into `ephemeralContext`. The server only strips stale profile blocks from conversation history before generation.

## References — Read These When the Task Calls for Them

The main file describes the protocol surface. Detailed material lives next door:

- `references/generate-response.md` — full `generate_response` schema, all event types, accumulation rules, stage-send rule, server-side `source` processing
- `references/message-format.md` — `[From: ...]` / `[msg:N]` format for every channel, `source` routing table, `channel_mappings` schema, group log file format
- `references/profile-matching.md` — frontmatter conventions per platform, the matching algorithm with pseudocode, 500-char limit, cross-platform identity
- `references/reply-quoting.md` — incoming `buildReplyContext` for each platform, outgoing reply references, multi-part response rules, edge cases

## The `generate_response` Message

Every bridge sends this to ask Alma for an AI reply:

```json
{
  "type": "generate_response",
  "data": {
    "threadId": "<uuid>",
    "model": "<model-id or undefined>",
    "userMessage": {
      "role": "user",
      "parts": [
        {"type": "text", "text": "<formatted message>"}
      ]
    },
    "ephemeralContext": "<assembled per-turn context>",
    "source": "<platform-identifier>"
  }
}
```

### The `source` Field

`source` is what tells the Alma server which channel-specific system prompt to apply and whether to strip stale ephemeral blocks from history.

| Source | Channel | Context |
|---|---|---|
| `"telegram"` | Telegram | Private/DM chat (`chatId > 0`) |
| `"telegram-group"` | Telegram | Group chat (`chatId < 0`) |
| `"discord"` | Discord | All messages (guild + DM) |
| `"feishu"` | Feishu | All messages |
| `"lark"` | Lark | International Feishu |
| `"weixin"` | WeChat | All messages |

For custom bridges, spoof Telegram: `"telegram-group"` for group chats, `"telegram"` for private chats. You inherit Telegram's group rules, history stripping, and privacy firewall for free. The full routing table including which sources trigger history stripping lives in `references/message-format.md`.

### Optional `data` Fields

| Field | Type | Purpose |
|---|---|---|
| `retryOfMessageId` | string | Retry a failed generation from this message |
| `replaceMessageId` | string | Replace an existing message in-place |
| `tools` | string[] | Override available tool keys |
| `reasoningEffort` | string | Reasoning effort level |
| `ephemeralModel` | string | Override model for this turn only |
| `noTools` | boolean | Disable all tools |
| `enabledMCPServerIds` | string[] | MCP servers to enable for this turn |
| `userMessageMetadata` | object | Metadata attached to the saved user message |

## Message Text Format

The text inside `userMessage.parts[0].text` follows the same pattern as built-in bridges.

### Telegram Private

```
[msg:42] actual message text
```

Built-in Telegram private chats do not include a `[From: ...]` header — only `[msg:N]`.

### Telegram Group

```
[From: DisplayName (@username) [id:123456789] [msg:42]] actual message text
```

`[msg:N]` is **inside** the `[From: ...]` header, not on a separate line. Use the **real platform message_id**, never a counter or hardcoded 0. `(@username)` is omitted when the platform does not have usernames, e.g. QQ/OneBot.

### With Reply / Quote

```
[From: DisplayName (@username) [id:...] [msg:43]] [Replying to Someone's message: "quoted text up to 200 chars"]
actual reply text
```

The `[Replying to X's message: "..."]` line is produced by `buildReplyContext()`. The full protocol — how to extract the reply target on each platform, what `get_msg` returns for OneBot, how to handle deleted quotes — lives in `references/reply-quoting.md`.

### Outgoing Reply Behaviour

Built-in bridges always reply to the triggering user message:

| Bridge | Mechanism | Field |
|---|---|---|
| Telegram | `reply_parameters: {message_id: userMessageId}` | In `sendMessage` |
| Discord | `messageReference: {messageId: originalId}` | In `createMessage` |
| QQ/OneBot | `reply` segment prepended to message array | First in `send_msg` |

Only the first chunk of a multi-part response carries the reply reference. The Alma OneBot bridge specifically consumes the reply on the first **successfully sent visible** message in the turn (stage text, tool-call status, or final reply); later visible messages in the same turn are plain. Built-in Telegram instead leaves stage text plain and attaches the reply to the final `sendReply`. The deviation is deliberate — see `references/reply-quoting.md`.

## EphemeralContext (Per-Turn System Prompt)

The bridge assembles `ephemeralContext` before each `generate_response`:

1. **SENDER PROFILE block** — injected by bridge-side profile matching (next section)
2. **PEOPLE PROFILES summary** — `PEOPLE PROFILES — You know N people. Use alma people list/show to look up profiles.`
3. **Recent group chat history** — last ~30 messages from the group log (group chats only)
4. **Cross-group context** — recent interactions from other groups (optional)

The server then layers SOUL.md, Memory, tool descriptions, and the channel-specific system prompt on top, based on `source`.

### Server-Side History Stripping

When `source` is `"telegram"`, `"telegram-group"`, `"discord"`, or `"feishu"`, the server strips stale ephemeral content from **older** user messages in conversation history:

- `[SENDER PROFILE]` blocks
- `RECENT GROUP CHAT HISTORY` blocks
- `GROUP-SPECIFIC RULES` blocks
- `PEOPLE PROFILES` blocks

The **last** user message keeps its full ephemeral context. This prevents stale profiles from accumulating in long conversations.

`"weixin"` is intentionally **not** in this list — WeChat messages do not get history stripping.

## SENDER PROFILE Matching (Bridge-Side)

This is the most consequential implementation for custom bridge authors. If you get nothing else right, get this right.

### Algorithm

1. Bridge receives a message with a sender ID (e.g. QQ user ID `123456789`).
2. Bridge scans every `.md` file in `~/.config/alma/people/`.
3. For each file, parse the YAML frontmatter and look for a matching platform ID.
4. If a match is found **and** the file is under 500 characters, inject it into `ephemeralContext` as a `[SENDER PROFILE]` block.

### Platform-Specific Matching

| Bridge | Frontmatter Field | Fallback |
|---|---|---|
| Telegram | `telegram_id: "<id>"` | Filename matches display name (case-insensitive) |
| Discord | `discord_id: "<id>"` or `discord_username: "<name>"` | Filename matches display name |
| Feishu | `feishu_id: "<id>"` | Filename matches display name |
| QQ (custom) | `qq_id: "<id>"` | Filename matches display name |

### Injected Format

```
[SENDER PROFILE — alice]:
---
telegram_id: "123456789"
username: alice_wonder
---
Alice is a designer based in Tokyo. Prefers concise answers.
[/SENDER PROFILE]
```

### Cross-Platform Identity

Include multiple platform IDs in the same profile so one file matches across bridges:

```yaml
---
telegram_id: "123456789"
qq_id: "123456789"
discord_id: "987654321"
username: starlight
---
```

Name profile files by a stable ID, not a nickname — different people can share display names across groups. Full matching pseudocode and edge cases live in `references/profile-matching.md`.

## The `channel_mappings` Table

Maps platform conversations to Alma threads.

```sql
CREATE TABLE channel_mappings (
    id TEXT PRIMARY KEY,
    platform TEXT NOT NULL,         -- "telegram" | "discord" | "feishu" | "lark" | "weixin"
    external_chat_id TEXT NOT NULL,
    external_user_id TEXT NOT NULL,
    thread_id TEXT NOT NULL REFERENCES chat_threads(id) ON DELETE CASCADE,
    is_active INTEGER DEFAULT 1,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL
);
```

**Lookup key**: `(platform, external_chat_id, external_user_id)`.

**Platform `userId` conventions**:

| Channel | Private Chat | Group Chat |
|---|---|---|
| Telegram | Sender's Telegram ID | `"group"` |
| Discord | Sender's Discord user ID | Sender's Discord user ID |
| Feishu | Sender's Feishu user ID | `"group"` |
| Weixin | chat ID (= `external_chat_id`) | N/A |
| QQ (custom) | Sender's QQ ID | `"group"` |

**Outgoing routing** (`detectPlatformForChatId`): `discord` → Discord, `weixin` → WeChat, **everything else → Telegram (default fallback)**. This describes Alma's built-in process boundary. **An external reverse-WS bridge must not write `channel_mappings` directly** — Alma keeps `chat_threads.db` locked while running. Keep your own session-to-thread mapping in the bridge's own database and create Alma threads through REST. The full schema and routing rationale live in `references/message-format.md`.

## Building a Custom Bridge — Checklist

### Get for Free by Spoofing `"telegram"` / `"telegram-group"`

- Server-side ephemeral context stripping for stale SENDER PROFILE blocks
- Telegram group chat system prompt (privacy firewall, people observation, frequency control)
- People profile CLI integration (`alma people show/list/append`)
- Telegram-style server prompt behaviour driven by `source`, message text, and `ephemeralContext`

### Must Implement Yourself

- **SENDER PROFILE scanning and injection** — scan `people/*.md` for your platform's ID field, inject the matched profile under 500 chars
- **Message text formatting** — `[From: name [id:xxx] [msg:N]]` prefix with the **real** message ID
- **Reply/quoting context** — `buildReplyContext` for incoming quotes; reply segment for outgoing
- **Response delivery** — listen on the Alma WS for `message_delta` / `message_updated`; send visible text before tool calls as **stage messages**
- **Optional tool-call display** — if enabled, emit plain progress messages like `正在调用工具：WebSearch\n参数：{"query":"..."}` after tool input is available; never treat these as final assistant text
- **People profile auto-creation** (recommended) — create `.md` files for new senders with platform IDs in frontmatter
- **Group chat log** — write logs to `~/.config/alma/groups/<chatId>_<YYYY-MM-DD>.log` so `alma group list/history/search/context` works
- **Active sends for custom platforms** — `alma group send` is Telegram-only; expose a bridge-owned HTTP endpoint or tool and inject its usage into `ephemeralContext`

### Implementation Steps

1. **Connect** to `ws://127.0.0.1:23001/ws/threads`.
2. **Resolve or create the Alma thread** through Alma REST. Keep platform session → thread mapping in your own database.
3. **Scan people profiles** for the sender's platform ID → build the SENDER PROFILE block.
4. **Format the message** — groups: `[From: ... [id:N] [msg:N]]`; private: `[msg:N]`. Prepend reply context if applicable.
5. **Send** `generate_response` with `source = "telegram-group"` for groups, `"telegram"` for private.
6. **Collect the response** from `message_delta` events (`partType: "text"` / `"output_text"`, `text-delta`, and `part_add` text parts).
7. **Send stage text** when `part_add` introduces a tool invocation. Do not treat the immediate `generation_completed` / `thread_generating=false` after a tool invocation as final.
8. **Optionally send tool-call status** after `tool_input_append` has supplied parameters. Keep it separate from stage / final reply text.
9. **Send final remaining text** to the platform. Attach the reply reference / `@`-mention rules per platform conventions above.
10. **Handle bidirectional forwarding** — use `message_updated` (not `message_added`, which has empty text for assistant messages) and filter by generation state.

## Response Events (Server → Bridge)

| Event Type | Purpose | Key Fields |
|---|---|---|
| `message_delta` | Streaming text chunk and tool boundary | `threadId`, `deltas[].text`, `deltas[].partType`, `deltas[].part` |
| `message_updated` | Message state change | `threadId`, `messageId`, `role`, `text` (full), `isGenerating` |
| `message_added` | New message saved | `threadId`, `messageId`, `role`, `text` (**empty for assistant!**) |
| `generation_completed` | Generation finished | `threadId`, `itemId` |
| `thread_generating` | Generation state toggle | `threadId`, `isGenerating` |
| `tool_status` | Tool execution update | `threadId`, `toolName`, `status` |
| `error` | Error occurred | `threadId`, `error` |

**Critical**: `message_added` fires with empty text for assistant messages because the shell is created before deltas arrive. Use `message_updated` for bidirectional forwarding, and filter by `thread_generating { isGenerating: false }` to get the final text. The full event sequence and accumulation rules — including the stage-send rule at tool boundaries — live in `references/generate-response.md`.

## Quick Reference: Full Message Flow

```
 1. Bridge connects to ws://127.0.0.1:23001/ws/threads
 2. Platform message arrives (e.g. QQ group, user 123456789, message_id=42)
 3. External bridges resolve session_key → threadId in their own DB and create
    Alma threads through REST when needed. Do not write Alma's channel_mappings.
 4. Bridge scans ~/.config/alma/people/*.md for qq_id match → SENDER PROFILE block.
 5. If the message has a reply/quote:
    - Fetch the quoted message (e.g. OneBot get_msg)
    - Build [Replying to X's message: "..."] context
 6. Bridge assembles ephemeralContext = SENDER PROFILE + PEOPLE PROFILES summary + history.
 7. Bridge sends generate_response with source="telegram-group" and the formatted text.
 8. Server streams message_delta events. Bridge accumulates visible text and emits
    stage text before each tool invocation.
 9. If enabled, bridge sends a separate tool-call status message after parameters arrive.
10. Bridge sends final remaining response. Reply reference attaches to the first visible
    bridge message in the turn (QQ); final message is plain if a stage/tool message
    already consumed it.
11. Bridge writes group chat logs to ~/.config/alma/groups/<chatId>_<YYYY-MM-DD>.log
    so alma group list/history/search/context can read them.
```
