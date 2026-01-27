# CLAUDE.md

This file provides guidance to Claude Code when working with code in this repository.

## Repository Structure

This is a Claude Code plugin marketplace hosting curated skills for Protocol Buffers development.

```
.claude-plugin/marketplace.json       # Marketplace metadata
plugins/
  protobuf/
    .claude-plugin/plugin.json        # Plugin config, version, keywords
    skills/
      protobuf/                       # Skill directory
        SKILL.md                      # Main skill (loaded when triggered)
        assets/                       # Templates: buf.yaml, buf.gen.*.yaml
        references/                   # Detailed docs (loaded on demand)
```

Skill triggers: `*.proto`, `buf.yaml`, `buf.*.yaml`, `buf.gen.yaml`, `buf.gen.*.yaml`, `buf.lock`

## Plugin Development

### Skill Structure

Each skill follows the [Agent Skills](https://agentskills.io) open standard:
- `SKILL.md`: Entry point with frontmatter and instructions
- `assets/`: Reusable templates Claude can copy directly
- `references/`: Progressive disclosure docs (loaded only when needed)

### Key Principles (Anthropic best practices)

1. **Keep SKILL.md under 500 lines** - Move details to reference files
2. **Descriptions matter** - Claude uses them to decide when to load the skill
3. **Progressive disclosure** - SKILL.md links to detailed references
4. **Concise is key** - Every token competes with conversation context
5. **Complete examples as assets** - Holistic proto files go in assets/
6. **Small patterns inline** - Fragment examples stay in reference docs

### Testing Changes

1. Open a fresh Claude Code session
2. Open a `.proto`, `buf.yaml`, or `buf.gen.yaml` file to trigger the skill
3. Verify the skill loads and provides expected guidance
4. Test common tasks (create message, create service, add field, fix lint error, configure codegen)

## Code Style

- YAML files: 2-space indent
- Markdown: One sentence per line (improves diffs)
- Proto examples in assets/: Must pass `buf lint` and `buf format`
