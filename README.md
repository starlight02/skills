# Skills

Reusable skills for Claude / Codex / Alma agents. Each directory is a self-contained skill
with a `SKILL.md` (the trigger description + core workflow) and a `references/` directory
of deeper material that the agent loads on demand.

## Included Skills

| Skill | Use it for |
| --- | --- |
| [`alma-channel-protocol`](alma-channel-protocol/) | Building or debugging custom Alma channel bridges that talk to Alma over WebSocket — protocol reference for `generate_response`, SENDER PROFILE matching, reply/quoting. |
| [`alma-extensibility`](alma-extensibility/) | Choosing and using Alma extension points: REST API, CLI, plugins, skills, MCP servers, People Profiles, and custom bridge patterns. |
| [`alma-onebot-bridge`](alma-onebot-bridge/) | Building or extending the Rust smol + Trillium OneBot v11 reverse WebSocket bridge that connects Alma to QQ; includes macOS menu bar app and Windows tray app packaging. |
| [`macos-admin-permission`](macos-admin-permission/) | Running approved macOS admin operations through the native GUI password prompt with `osascript ... with administrator privileges`, instead of relying on `sudo`. Platform-agnostic — not coupled to any specific app. |

## Repository Layout

Every skill follows the same shape, designed around the
[progressive disclosure model](https://github.com/anthropics/skills) for agent skills:

```text
skill-name/
├── SKILL.md          name + description (always in context) + workflow (loaded when triggered)
└── references/       supporting material, loaded on demand via links from SKILL.md
    ├── topic-a.md
    └── topic-b.md
```

- `SKILL.md` carries YAML frontmatter (`name`, `description`) plus the main body. The body
  is kept under ~500 lines and links into `references/` for detail.
- `references/*.md` files are read only when the active task touches them, so the metadata
  footprint stays small.

## Installation

Install the skills directly from this repository:

```bash
npx skills add https://github.com/starlight02/skills
```

Restart the agent or reload skills after installation if your client requires it.

## Contributing

When adding or editing a skill:

1. Keep `SKILL.md` under ~500 lines. Move long protocol references, schema dumps, and
   worked examples into `references/`.
2. Write the `description` field to trigger reliably. Be specific about use cases, list
   relevant keywords the user might type, and explain *when* to invoke the skill — not
   just *what* it does.
3. Explain the **why** behind instructions. Heavy MUSTs tend to backfire; principles
   travel better than rigid rules.
4. Verify the content against the live system the skill describes. Outdated steps cost
   more debugging time than no steps at all.
