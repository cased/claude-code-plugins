# Contributing to Claude Code Plugins

We welcome plugin contributions! This guide explains how to submit plugins to the marketplace.

## Plugin Requirements

### Structure

Every plugin must include:

```
plugin-name/
├── .claude-plugin/
│   └── plugin.json       # Required: Plugin manifest
├── README.md             # Required: Documentation
└── [components]          # At least one of: skills/, commands/, hooks/, .mcp.json
```

### plugin.json Manifest

```json
{
  "name": "plugin-name",
  "version": "1.0.0",
  "description": "What the plugin does",
  "author": {
    "name": "Your Name",
    "url": "https://example.com"
  },
  "license": "MIT",
  "keywords": ["keyword1", "keyword2"],
  "skills": "./skills",
  "commands": "./commands",
  "hooks": "./hooks",
  "mcpServers": "./.mcp.json"
}
```

**Required fields:** `name`, `version`, `description`

**Optional fields:** `author`, `homepage`, `repository`, `license`, `keywords`, component paths

### Naming Conventions

- Plugin name: lowercase with hyphens (`my-plugin`)
- Skill names: lowercase with hyphens, max 64 characters
- Command files: lowercase with hyphens (`.md` extension)

## Submission Process

1. **Fork this repository**

2. **Create your plugin** in `plugins/your-plugin-name/`

3. **Update marketplace.json** - Add your plugin to the `plugins` array:
   ```json
   {
     "name": "your-plugin-name",
     "path": "./plugins/your-plugin-name",
     "description": "Brief description",
     "version": "1.0.0",
     "type": "skill|command|hook|mcp|mixed"
   }
   ```

4. **Test your plugin** - Verify it installs and works correctly

5. **Submit a Pull Request** with:
   - Clear description of what the plugin does
   - Installation and usage instructions in your README
   - Any prerequisites or dependencies

## Quality Guidelines

### Skills

- Focused scope (one capability per skill)
- Description includes trigger phrases
- Concrete examples with input/output
- Progressive disclosure (reference external docs for details)

### Commands

- Clear description in YAML frontmatter
- Document any required arguments
- Include usage examples

### Hooks

- Document trigger conditions
- Specify timeout requirements
- Handle errors gracefully

### MCP Servers

- Document required environment variables
- Include setup instructions
- Specify any external dependencies

## Review Criteria

- **Usefulness**: Does it solve a real problem?
- **Quality**: Is it well-documented and tested?
- **Security**: No credentials, safe operations
- **Originality**: Doesn't duplicate existing plugins

## License

By contributing, you agree that your plugin will be licensed under the same license as this repository (MIT).
