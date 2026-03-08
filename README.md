# Ink Cloud Skill

Deploy and manage cloud services on [Ink](https://ml.ink) directly from Claude Code.

## Install

```bash
claude install-plugin mldotink/ink-skill
```

Or manually:

```bash
# Add as a project plugin
git clone https://github.com/mldotink/ink-skill .claude/plugins/ink-skill

# Or add as a global plugin
git clone https://github.com/mldotink/ink-skill ~/.claude/plugins/ink-skill
```

## Setup

1. Get an API key from the [Ink dashboard](https://ml.ink/onboarding) or under Settings > Agent Keys
2. Set the environment variable:
   ```bash
   export INK_API_KEY=dk_live_your_key_here
   ```

## Usage

Once installed, just ask Claude to deploy, manage services, or check your infrastructure:

- "Deploy my app to Ink"
- "List my services"
- "Create a database and deploy my app with it"
- "Add a custom domain to my service"
- "Deploy my frontend and backend as a full-stack app"

## Structure

```
ink-skill/
├── .claude-plugin/
│   ├── plugin.json         # Plugin manifest
│   └── marketplace.json    # Marketplace listing
├── skills/
│   └── ink/
│       └── SKILL.md        # Deployment instructions
├── CLAUDE.md               # Plugin context
└── README.md
```

## Alternative: MCP Server

For a richer integration with tool-calling support, use the [Ink MCP Server](https://github.com/mldotink/ink-mcp) instead.
