# ink-skill

Deploy and manage cloud services on [Ink](https://ml.ink) directly from Claude Code.

## Install

```bash
# Add as a project skill
git clone https://github.com/mldotink/ink-skill .claude/skills/ink-skill

# Or add as a global skill
git clone https://github.com/mldotink/ink-skill ~/.claude/skills/ink-skill
```

## Setup

1. Get an API key from the [Ink dashboard](https://ml.ink) (Settings > API Keys)
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

## Alternative: MCP Server

For a richer integration with tool-calling support, use the [Ink MCP Server](https://github.com/mldotink/ink-mcp) instead.
