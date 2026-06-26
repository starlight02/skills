# generate_response Protocol Reference

This document is the complete WebSocket protocol reference for Alma's `generate_response`
message and the server-to-bridge response events.

## WebSocket Endpoint

```
ws://127.0.0.1:23001/ws/threads
```

All bridges (built-in and custom) connect here. The server accepts JSON messages.

## Request: `generate_response`

```json
{
  "type": "generate_response",
  "data": {
    "threadId": "<uuid>",
    "model": "<model-id or undefined>",
    "userMessage": {
      "role": "user",
      "parts": [
        {"type": "text", "text": "<formatted message>"},
        {"type": "file", "url": "data:image/jpeg;base64,...", "mediaType": "image/jpeg", "filename": "photo.jpg"}
      ]
    },
    "ephemeralContext": "<full system prompt string>",
    "source": "<platform-identifier>"
  }
}
```

### Required Fields

| Field | Type | Description |
|-------|------|-------------|
| `data.threadId` | string | Alma thread UUID (from `channel_mappings` or `POST /api/threads`) |
| `data.userMessage` | object | The user message with `role` and `parts` |
| `data.userMessage.parts` | array | Array of content parts (text, file) |

### Optional Fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `data.model` | string \| null | Server default | Model ID (e.g., `"anthropic:claude-sonnet-4-20250514"`) |
| `data.source` | string | none | Platform identifier (triggers source-specific processing) |
| `data.ephemeralContext` | string | `""` | Per-turn system prompt (SENDER PROFILE, group history, etc.) |
| `data.retryOfMessageId` | string | none | Retry a failed generation from this message |
| `data.replaceMessageId` | string | none | Replace an existing message in-place |
| `data.tools` | string[] | all | Override available tool keys |
| `data.reasoningEffort` | string | default | Reasoning effort level |
| `data.enabledMCPServerIds` | string[] | none | MCP servers to enable for this turn |
| `data.noTools` | boolean | false | Disable all tools |
| `data.ephemeralModel` | string | none | Override model for this turn only |
| `data.userMessageMetadata` | object | none | Metadata attached to saved user message |
| `data.fromQuickChat` | boolean | false | Quick chat mode flag |
| `data.hummingbirdContext` | object | none | Hummingbird context data |

### `userMessage.parts` Content Types

| Type | Fields | Purpose |
|------|--------|---------|
| `text` | `text: string` | Plain text message content (includes `[From:]` prefix, `[msg:N]` tags) |
| `file` | `url: string`, `mediaType: string`, `filename: string` | Image/file attachment (supports `data:` URIs for base64) |
| `step-start` | none | Internal: marks start of a generation step |
| `reasoning` | `text: string` | Internal: model thinking/reasoning content |

Only `text` and `file` types should be sent by bridges. `step-start` and `reasoning` are
server-internal.

## Response Events (Server → Bridge)

### Event Sequence During Generation

The complete event sequence observed from a `generate_response` call:

```
 1. thread_created                          Thread created (new threads only)
 2. message_added      (user, text)         User message saved to DB
 3. message_updated    (user)               User message state updated
 4. skill_analysis_progress                 Skill analysis phase
 5. thread_generating  {isGenerating: true} Generation started
 6. message_added      (assistant, EMPTY!)  Assistant message shell created
 7. message_updated    (assistant, partial) Partial text during generation
 8. message_delta      (multiple)           Streaming text chunks
 9. generation_completed                    Generation finished
10. thread_generating {isGenerating: false} Generation ended
11. context_usage_update                     Token usage report
12. message_updated   (assistant, FULL)      Final complete text
```

### `message_delta` — Streaming Text Chunks

```json
{
  "type": "message_delta",
  "data": {
    "threadId": "...",
    "deltas": [
      {"type": "text_append", "partType": "text", "text": "chunk of text"},
      {"type": "text_append", "partType": "reasoning", "text": "...thinking..."}
    ]
  }
}
```

**Accumulation rule**: Accumulate visible text from `text_append` where `partType` is
`"text"` or `"output_text"`, from `text-delta`, and from `part_add` text parts. Ignore
`partType == "reasoning"` (model thinking/internal).

**Stage-send rule used by built-in bridges**: when a `part_add` delta starts a tool
invocation (`part.type == "tool-invocation"` or `part.type` starts with `"tool-"` and is
not a result part), send the visible text accumulated before that tool call as a plain
platform message. Keep the generation pending; the `generation_completed` and
`thread_generating=false` events immediately after a tool invocation are not final. When
the final assistant text arrives, strip the successfully sent visible stage prefix and send
only the remaining text. External bridges should treat the generation timeout as an idle
timeout and reset the wait window after each stage/tool-call progress event, since the
following tool call can take much longer than the initial text generation.

Custom bridges may expose an optional tool-call display setting. When enabled, emit a plain
platform message after the stage text for each tool invocation, for example
`正在调用工具：WebSearch\n参数：{"query":"alma"}`. Alma can emit the initial tool `part_add`
while `input` is still `{}` and then stream parameters through `tool_input_append`; custom
bridges should accumulate those deltas and wait until parameters are available or the tool
state leaves `input-streaming` before sending a parameter preview. This is a progress
notification only; do not count it as assistant response text when stripping the final
reply prefix.

### `message_updated` — Message State Change

```json
{
  "type": "message_updated",
  "data": {
    "threadId": "...",
    "messageId": "...",
    "role": "assistant",
    "text": "full or partial message text",
    "isGenerating": true
  }
}
```

Fires multiple times per message:
- During generation: `text` is partial, `isGenerating` is `true`
- After completion: `text` is the final complete text, `isGenerating` is `false`

### `message_added` — New Message Saved

```json
{
  "type": "message_added",
  "data": {
    "threadId": "...",
    "messageId": "...",
    "role": "assistant",
    "text": ""
  }
}
```

**Critical pitfall**: `text` is **always empty** for assistant messages. The message shell
is created before the AI starts generating. Do NOT use this event to capture assistant
replies — use `message_updated` or `message_delta` instead.

### `thread_generating` — Generation State Toggle

```json
{
  "type": "thread_generating",
  "data": {
    "threadId": "...",
    "isGenerating": true
  }
}
```

Use this to track whether a thread is currently generating. Essential for:
- Filtering `message_updated` events (only forward after `isGenerating: false`)
- Preventing duplicate forwarding during active generation
- UI indicators (typing animation, etc.)

### `generation_completed`

```json
{
  "type": "generation_completed",
  "data": {
    "threadId": "...",
    "itemId": "..."
  }
}
```

Signals that generation is finished. The final text is available in the last `message_updated`
event (with `isGenerating: false`).

### Other Events

| Event | Purpose |
|-------|---------|
| `tool_status` | Tool execution update: `{threadId, toolName, status}` |
| `error` | Error: `{threadId, error}` |
| `thread_updated` | Thread metadata changed (title, etc.) |
| `context_usage_update` | Token usage: `{threadId, usage}` |
| `skill_analysis_progress` | Skill analysis phase |

## Accumulating the Response

The recommended pattern for collecting the full AI response:

```
1. On generate_response sent: mark thread as "generating"
2. On message_delta visible text deltas: append text to buffer
3. On message_delta tool-invocation part_add: send the unsent visible prefix as a stage message
4. Reset the generation idle timeout after each stage message
5. On completion immediately after a tool invocation: keep waiting for continuation
6. On final completion with no pending tool continuation: mark thread as "done"
7. Strip <think>...</think> blocks from accumulated text
8. Send only the remaining text after successfully sent stage prefixes
```

Alternative: accumulate only from `message_delta` text_append events. This works but may
miss edge cases where the server modifies text after streaming (e.g., tool result insertion).

## Server-Side Source Processing

When the server receives `generate_response`, it checks `source`:

```javascript
if (source && ["telegram", "telegram-group", "discord", "feishu"].includes(source)) {
  // Strip stale ephemeral content from OLDER user messages in history:
  // - [SENDER PROFILE] blocks
  // - RECENT GROUP CHAT HISTORY blocks
  // - GROUP-SPECIFIC RULES blocks
  // - RECENT GROUP INTERACTIONS blocks
  // - PEOPLE PROFILES blocks
  // The LAST user message keeps its full ephemeral context.
}
```

**Important**: `"weixin"` is NOT included — WeChat messages do not get history stripping.

Additionally, the server appends channel-specific system prompt sections:

| Source | Added System Prompt |
|--------|-------------------|
| `"telegram"` | Telegram formatting rules, file/sticker/voice sending, message links |
| `"telegram-group"` | All of above + group chat rules (privacy, observation, frequency control) |
| `"discord"` | Discord markdown, channel directory, group behavior |
| `"feishu"` | Feish-specific rules (mostly handled via ephemeralContext from bridge) |
| `"weixin"` | WeChat file/voice sending, plain text formatting rules |
