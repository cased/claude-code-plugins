# Claude Code Plugins

A collection of plugins for [Claude Code](https://claude.com/claude-code) including skills, MCP servers, hooks, and slash commands.

## Available Plugins

| Plugin | Type | Description |
|--------|------|-------------|
| [kit-cli](./plugins/kit-cli) | skill | Run focused `kit` CLI commands for fast codebase exploration |

## Installation

### Using the `/plugin` Command

1. Add this marketplace to Claude Code:
   ```
   /plugin marketplace add cased/claude-code-plugins
   ```

2. Install a plugin:
   ```
   /plugin install kit-cli
   ```

### Manual Installation

Alternatively, clone and copy files directly:

```bash
git clone https://github.com/cased/claude-code-plugins.git
cp -r claude-code-plugins/plugins/kit-cli/skills/kit-cli ~/.claude/skills/
```

## Plugin Structure

Each plugin follows the Claude Code plugin specification:

```
plugin-name/
├── .claude-plugin/
│   └── plugin.json       # Plugin manifest
├── skills/               # Skills (optional)
│   └── skill-name/
│       ├── SKILL.md
│       └── reference.md
├── commands/             # Slash commands (optional)
├── hooks/                # Hooks (optional)
├── .mcp.json            # MCP server config (optional)
└── README.md            # Plugin documentation
```

## Contributing

See [CONTRIBUTING.md](./CONTRIBUTING.md) for guidelines on submitting plugins.

## License

MIT
