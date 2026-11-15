# Contributing

## No plugins are published

The marketplace in `.claude-plugin/marketplace.json` lists no plugins. The last one,
`poneglyph-intelligence`, was retired because the MCP endpoint it called was removed.

A new plugin goes under `plugins/<plugin-name>/` and gets an entry in the marketplace's `plugins` list.

## Commit Messages

This project uses [Conventional Commits](https://www.conventionalcommits.org/).

Use these commit prefixes:
- `feat:` — New features
- `fix:` — Bug fixes
- `chore:` — Maintenance tasks
