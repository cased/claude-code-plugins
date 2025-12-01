# Claude Code Plugins

A collection of plugins for [Claude Code](https://claude.com/claude-code) including skills, MCP servers, hooks, and slash commands.

## Available Plugins

| Plugin | Type | Description |
|--------|------|-------------|
| [kit-cli](./plugins/kit-cli) | skill | Run focused `kit` CLI commands for fast codebase exploration |

## Installation

### Manual Installation

1. Clone this repository:
   ```bash
   git clone https://github.com/cased/claude-code-plugins.git
   ```

2. Copy the desired plugin components to your Claude Code config:

   **Skills:**
   ```bash
   cp -r claude-code-plugins/plugins/kit-cli/skills/kit-cli ~/.claude/skills/
   ```

   **MCP Servers:** Copy the `.mcp.json` config or merge with your existing configuration.

   **Hooks:** Copy hook scripts to `~/.claude/hooks/` and update `~/.claude/settings.json`.

   **Commands:** Copy command files to `~/.claude/commands/`.

3. Restart Claude Code to load the new plugins.

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
