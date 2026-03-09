# Ink Skills

Agent skill for [Ink](https://ml.ink) — a cloud platform designed for AI agents to deploy and manage services autonomously. It makes deployments simple enough that fully autonomous agents can handle the entire lifecycle: create, deploy, monitor, and scale services without human intervention.

## Installation

```bash
npx skills add mldotink/ink-skill
```

### Claude Code plugin marketplace

```
/plugin install ink@ink-skill
```

### Manual

```bash
claude install-plugin mldotink/ink-skill
```

## Setup

1. Get an API key from the [Ink dashboard](https://ml.ink/onboarding) or under Settings > Agent Keys
2. Set the environment variable:
   ```bash
   export INK_API_KEY=dk_live_your_key_here
   ```

Or run `ink login` after installing the CLI.

## Skill surface

This repo ships one installable skill:

- [`ink`](skills/ink/SKILL.md) — Deploy apps, manage services, databases, DNS, custom domains, and workspaces on Ink

The skill uses the [Ink CLI](https://github.com/mldotink/cli) (`@mldotink/cli`) and will prompt to install it if missing.

## Usage

Once installed, just ask your agent to deploy or manage infrastructure:

- "Deploy my app to Ink"
- "List my services"
- "Create a database and deploy my app with it"
- "Add a custom domain to my service"
- "Deploy my frontend and backend as a full-stack app"
- "Scale my service to 1Gi memory"

## Repository structure

```
ink-skill/
├── .claude-plugin/
│   ├── plugin.json
│   └── marketplace.json
├── skills/
│   └── ink/
│       └── SKILL.md
├── CLAUDE.md
└── README.md
```
