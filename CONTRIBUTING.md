# Contributing

## The only plugin is `poneglyph-intelligence`

All changes go into `plugins/poneglyph-intelligence/`. The skill lives at `skills/poneglyph/SKILL.md` and the MCP config at `.mcp.json`.

After changing the skill locally, sync it to your Claude Code installation:
```bash
cp plugins/poneglyph-intelligence/skills/poneglyph/SKILL.md ~/.claude/skills/poneglyph/SKILL.md
```

## Commit Messages

This project uses [Conventional Commits](https://www.conventionalcommits.org/) and [release-please](https://github.com/googleapis/release-please) for automated releases.

Use these commit prefixes:
- `feat:` — New features (bumps minor version)
- `fix:` — Bug fixes (bumps patch version)
- `chore:` — Maintenance tasks (no version bump)

Always use `poneglyph-intelligence` as the scope when changing the plugin:
```bash
git commit -m "feat(poneglyph-intelligence): add signals use case for job openings"
git commit -m "fix(poneglyph-intelligence): correct pagination example"
git commit -m "chore: update README"
```

## Releases

Releases are automated via release-please once the `GH_PAT_RELEASE_PLEASE_ACTION` secret is active:

1. Push commits to `main` using conventional commit messages
2. Release-please opens a Release PR with changelog and version bump
3. Merge the Release PR to publish the new version

The version in `.claude-plugin/plugin.json` is automatically updated on release.
