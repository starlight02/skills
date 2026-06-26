---
name: alma-extensibility
description: >-
  Complete guide to Alma's extensibility surface — REST API on `localhost:23001`, the
  `alma` CLI (`alma run`, `alma config`, `alma thread`, `alma group`, `alma memory`,
  `alma soul`, `alma people`, etc.), TypeScript plugin hooks, the SKILL.md-based skill
  system, MCP servers, People Profiles, and the two patterns for integrating unsupported
  chat platforms. Use this skill whenever the user wants to: build an Alma integration,
  write or debug an Alma plugin, create or edit a SKILL.md, register an MCP server,
  connect an unsupported messaging platform (QQ/OneBot, Slack, Matrix, Discord-without-
  bot-token), call the Alma REST API from a script, automate Alma via CLI, ask what Alma
  can/can't do programmatically, ask why a plugin hook doesn't fire, ask which extension
  point fits a use case, write a custom channel bridge, manage People Profiles, configure
  `~/.config/alma/mcp.json`, or anything else that involves extending Alma without
  modifying its source. Also trigger when the user mentions "alma run", "alma config",
  "alma hooks", "alma plugin", "alma skill", "alma mcp", "alma channel", or asks "is X
  possible with Alma" — prefer consulting this skill before answering, even when the
  user does not name it directly.
---

# Alma Extensibility Guide

Reference for building integrations, plugins, and extensions for Alma. Most extensibility questions resolve to "which of these five extension points fits the job?" — this skill is the map; the references files are the details.

## Core Architecture

Alma's **channel system is closed** — Telegram, Discord, WeChat, Feishu are built into the binary with no registration API. But there are **five extension points** that cover almost every real need:

| Point | Mechanism | Best For |
|---|---|---|
| **REST API** | `http://localhost:23001/api/*` | External processes managing Alma state |
| **`alma run`** | CLI with the full AI pipeline | Getting AI replies from scripts and bridges |
| **Plugins** | TypeScript in `~/.config/alma/plugins/` | Hooking into message and tool lifecycle |
| **Skills** | SKILL.md prompt templates | Teaching the AI new capabilities |
| **MCP Servers** | External tool servers | Providing structured tools |

Plus the cross-cutting **People Profiles** mechanism (`~/.config/alma/people/`) that all five extension points can read.

## References — Read These When the Task Calls for Them

- `references/rest-api.md` — endpoint table, payload rules, settings-update gotchas, usage snippets
- `references/cli-reference.md` — every `alma` subcommand and flag, with environment variables
- `references/plugin-system.md` — manifest schema, `PluginContext` API, all hook events, hard limits, example plugin
- `references/skills-and-mcp.md` — SKILL.md frontmatter, progressive disclosure, MCP transport types and config
- `references/people-profiles.md` — frontmatter conventions, cross-platform identity, 500-char SENDER PROFILE limit, auto-creation
- `references/channel-bridge.md` — WebSocket vs REST+CLI bridge patterns, comparison table, what works and what doesn't
- The **`alma-channel-protocol`** skill — authoritative WS protocol for custom channel bridges

## 1. REST API

**Base URL:** `http://localhost:23001` | **Full spec:** `~/.config/alma/api-spec.md`

Covers: settings, AI providers, models, threads. Important rules to internalize:

- **Settings PUT** requires the complete object — partial updates fail. Always GET first, mutate, then PUT.
- **Model IDs** are always `providerId:modelId` (e.g. `abc123:gpt-4o`).
- **API keys** are encrypted in storage and never echoed in responses.
- **WebSocket sync** broadcasts API changes to connected GUI clients in real time.

Endpoint table, schemas, and worked examples are in `references/rest-api.md`.

## 2. CLI

Run `alma help` for the full list. `ALMA_API_URL` overrides the base URL (default `http://localhost:23001`).

The CLI covers configuration & providers, threads & projects, AI completion (`alma run`), memory & identity (`alma memory`, `alma soul`, `alma emotion`), skills & tools & cron, media & messaging, browser & desktop automation, activity recording, and data management.

The single most important command for integrations is `alma run`:

```bash
alma run [prompt]                        # Full pipeline: SOUL + Memory + Tools + Plugins
  -m, --model <provider:model>           # Override model
  -s, --system <prompt>                  # Prepend system instruction
  --raw                                  # Strip markdown
  --no-stream                            # Buffer, print at end

# Target a specific thread (critical for bridge services):
ALMA_THREAD_ID=<id> alma run "hello"
```

The complete subcommand and flag reference lives in `references/cli-reference.md`.

## 3. Plugin System

Plugins are **TypeScript extensions running inside Alma's process**. They hook into message and tool lifecycle events.

**Good fits**: forwarding messages, modifying prompts before the AI sees them, logging tool usage, reacting to AI replies, adding UI commands.

**Directory**: `~/.config/alma/plugins/<plugin-name>/`

Key hook events:

| Event | Trigger | Capability |
|---|---|---|
| `chat.message.willSend` | Before the user message reaches the AI | **Mutable** — modify prompt text or model |
| `chat.message.didReceive` | After the AI generates a response | React to replies, e.g. forward to an external channel |
| `tool.willExecute` | Before a tool runs | Log, block, or modify tool arguments |
| `tool.didExecute` | After a tool completes | Analytics, telemetry |
| `tool.onError` | A tool execution fails | Error monitoring, retry logic |
| `thread.activated` | User switches to a thread | Engagement tracking, analytics UI |

Hard limits worth knowing before you start:

- Cannot create inbound messages — there is no `chat.message.create`. Plugins react; they do not originate.
- Cannot register channel adapters — channels live in the binary.
- Cannot modify the AI pipeline internals — only `willSend` / `didReceive` on the user side.
- Runs in Alma's process — uncaught exceptions can crash the app. Wrap handlers in try/catch.

`manifest.json` schema, the `PluginContext` API, permissions table, and an end-to-end example live in `references/plugin-system.md`.

## 4. Skill System

Skills are **prompt templates** loaded into the AI's context. They teach the agent new capabilities using natural-language instructions and access to Bash / file tools.

```
~/.config/alma/skills/<name>/
├── SKILL.md              # required: YAML frontmatter + markdown
├── scripts/              # optional: executable helpers
├── references/           # optional: docs loaded on demand
└── assets/               # optional: templates, images, data files
```

Design principles that matter:

1. **The description drives triggering.** The AI reads name + description to decide whether to load the skill. Be specific about use cases and trigger phrases.
2. **Keep SKILL.md under ~500 lines.** Use `references/` for detail; link to them from SKILL.md.
3. **Explain WHY, not just WHAT.** Today's models are smart enough to generalize from principles. Heavy-handed MUSTs tend to backfire.
4. **Include examples.** Concrete input/output beats abstract prose.

Frontmatter schema, `allowed-tools`, progressive disclosure mechanics, and management commands are in `references/skills-and-mcp.md`.

## 5. MCP Servers

External tool servers that expose structured capabilities to the AI. MCP tools appear alongside built-in tools and can be listed with `alma tool list`.

**Config:** `~/.config/alma/mcp.json`

```json
{
  "mcpServers": {
    "my-service": {
      "url": "http://127.0.0.1:8001/mcp/",
      "transport": "streamable-http"
    }
  }
}
```

Transport types (`streamable-http`, `sse`, `stdio`), authentication patterns, and error-handling semantics are in `references/skills-and-mcp.md`.

## 6. People Profiles

User profiles as markdown with YAML frontmatter, auto-loaded by `alma run` and by channel bridges.

```
~/.config/alma/people/<name>.md
```

```markdown
---
telegram_id: "123456789"
discord_id: "987654321"
qq_id: "10001000"
---
# Display Name

- Key facts, preferences, interaction history
```

All platform IDs must be **quoted strings** in YAML — unquoted large integers can lose precision in some parsers. Include every platform the person uses to enable cross-platform identity matching.

Field conventions, auto-creation patterns, file naming guidance, and the 500-character SENDER PROFILE size limit are in `references/people-profiles.md`. The bridge-side matching algorithm itself lives in the **`alma-channel-protocol`** skill.

## 7. Custom Channel Bridge Pattern

For integrating unsupported platforms (QQ/OneBot, Slack, Matrix, etc.), there are **two approaches** with different tradeoffs.

### Approach A: WebSocket Bridge (Native — Recommended)

Alma's built-in channels all use a WebSocket protocol to talk to the Alma server. A custom bridge using the same protocol gets **much better integration**.

```
External Platform → Custom Bridge → ws://127.0.0.1:23001/ws/threads → Alma Server
                    (generate_response, ephemeralContext, SENDER PROFILE injection)
```

**Advantages**: streaming responses, native SENDER PROFILE matching, reply/quoting support, the same processing pipeline as built-in channels, messages appear in the GUI with proper formatting.

For the WebSocket protocol itself (`generate_response`, event sequence, profile matching, reply/quoting), use the **`alma-channel-protocol`** skill — it is the authoritative reference.

### Approach B: REST API + `alma run` (External — Simpler)

For quick prototypes or when WebSocket integration is too much friction:

```
External Platform → Adapter → Bridge Service → REST API + alma run → Alma
```

Steps:

1. **Inbound** — bridge receives events from the external platform.
2. **Session mapping** — maintain `{platform}:{chatId}` → Alma Thread ID in your own file or DB.
3. **Thread creation** — `POST /api/threads {"title": "..."}`; the thread shows up in the GUI.
4. **AI completion** — `ALMA_THREAD_ID=<id> alma run --raw '<message>'`.
5. **Outbound** — call the external platform's API to send the reply.

See `references/rest-api.md` for thread endpoints and `references/cli-reference.md` for `alma run` flags. `references/channel-bridge.md` has the side-by-side comparison and the "what works / what doesn't" table.

### Comparison

| Feature | Approach A (WebSocket) | Approach B (REST + CLI) |
|---|---|---|
| Streaming output | Real-time deltas | Buffered, full response only |
| GUI integration | Native-level | Basic thread visibility |
| Profile matching | Automatic (SENDER PROFILE) | Manual (write profile files) |
| Reply/quoting | Supported | Not supported |
| Implementation effort | Higher (WebSocket protocol) | Lower (HTTP calls) |
| Dependencies | None (direct WS) | Requires `alma` CLI on PATH |

### What Works vs What Doesn't

| Works | Doesn't Work |
|---|---|
| Threads visible in GUI sidebar | No native channel settings panel |
| Full AI pipeline (SOUL + Memory + Tools + Plugins) | No channel registration API |
| Bidirectional messaging | No GUI ↔ external sync without a Plugin |
| Per-user/group conversation history | No channel identity badges |
| People Profiles auto-loaded | |

## 8. Decision Guide

| Goal | Use | Reference |
|---|---|---|
| Get an AI reply from an external script | `alma run` | `references/cli-reference.md` |
| Manage settings programmatically | REST API | `references/rest-api.md` |
| React to AI replies (forward, log) | Plugin (`didReceive`) | `references/plugin-system.md` |
| Modify prompts before the AI sees them | Plugin (`willSend`) | `references/plugin-system.md` |
| Monitor tool usage / errors | Plugin (`didExecute` / `onError`) | `references/plugin-system.md` |
| Teach the AI new capabilities | Skill (SKILL.md) | `references/skills-and-mcp.md` |
| Add tools from an external service | MCP Server | `references/skills-and-mcp.md` |
| Remember information about users | People Profiles | `references/people-profiles.md` |
| Connect an unsupported chat platform | Bridge (WebSocket, Approach A) | **`alma-channel-protocol`** skill |
| Quick prototype bridge | Bridge (REST + CLI, Approach B) | `references/channel-bridge.md` |
| Register a native channel adapter | Not possible | — |
