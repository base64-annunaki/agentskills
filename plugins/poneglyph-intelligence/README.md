# poneglyph-intelligence

Recruiting intelligence skill for the Poneglyph knowledge base. Search candidates, explore profiles, map connections between people and companies, enrich data from external sources, and manage signal watchers.

## Installation

```bash
claude plugin marketplace add base64-annunaki/agentskills
claude plugin install poneglyph-intelligence@poneglyph
```

## Setup

Set your personal API token permanently in Claude Code:

```bash
claude config set env.PONEGLYPH_API_TOKEN your-token-here
```

Generate your token at [app.pintl.ai](https://app.pintl.ai) → Settings → API Tokens.

## Usage

Once installed and configured, use `/poneglyph` in any Claude Code session to activate the recruiting intelligence skill.

## Updating

When a new version is released:

```bash
claude plugin marketplace update poneglyph
claude plugin update poneglyph-intelligence@poneglyph
```
