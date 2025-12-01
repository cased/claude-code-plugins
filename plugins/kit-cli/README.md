# kit-cli Plugin

A Claude Code skill for running focused `kit` CLI commands to build high-signal repository context fast.

## What it does

This skill enables Claude to use the [kit](https://github.com/cased/kit) CLI for:

- **File trees** - Structural overview of repositories
- **Symbol extraction** - Find functions, classes, and their locations
- **Code search** - Text and semantic search across codebases
- **Dependency graphs** - Map imports and module relationships
- **Context bundling** - Export slices of code for analysis

## Prerequisites

Install kit:

```bash
pipx install cased-kit
# or
pip install cased-kit
```

For semantic search, also install:

```bash
pip install "sentence-transformers>=2.6"
```

## Installation

Copy the skill to your Claude Code config:

```bash
cp -r skills/kit-cli ~/.claude/skills/
```

## Usage

Once installed, Claude will automatically use kit commands when you ask questions like:

- "What's the structure of this codebase?"
- "Find where `validateToken` is defined and used"
- "Show me the dependency graph for the Python modules"
- "Search for authentication-related code"

## Commands Reference

| Command | Purpose |
|---------|---------|
| `kit file-tree PATH` | Directory structure |
| `kit symbols PATH` | Extract functions/classes |
| `kit usages PATH Symbol` | Find symbol definitions and references |
| `kit search PATH "pattern"` | Text/regex search |
| `kit search-semantic PATH "query"` | Semantic code search |
| `kit dependencies PATH --language python` | Import/dependency graph |
| `kit context PATH file line` | Surrounding code window |

See the [reference.md](./skills/kit-cli/reference.md) for complete command documentation.

## License

MIT
