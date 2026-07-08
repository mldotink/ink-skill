# Ink Skills

Agent skill for [Ink](https://deployink.com) — a cloud platform designed for AI agents to deploy and manage services autonomously. It makes deployments simple enough that fully autonomous agents can handle the entire lifecycle: create, deploy, monitor, and scale services without human intervention.

## Installation

### Claude Code

```bash
npx skills add mldotink/ink-skill
```

Or via the Claude Code plugin manager:

```bash
claude plugin install ink
```

### OpenClaw

Copy the skill to your managed skills directory:

```bash
git clone https://github.com/mldotink/ink-skill /tmp/ink-skill-install
mkdir -p ~/.openclaw/skills
cp -r /tmp/ink-skill-install/skills/ink ~/.openclaw/skills/ink
rm -rf /tmp/ink-skill-install
```

Or add the repo as an extra skills directory in `~/.openclaw/openclaw.json`:

```json
{
  "skills": {
    "load": {
      "extraDirs": ["/path/to/ink-skill/skills"]
    }
  }
}
```

## Setup

1. Get an API key from the [Ink dashboard](https://deployink.com/onboarding) or under Settings > Agent Keys
2. Install the Ink CLI if you do not already have it:
   ```bash
   npm install -g @mldotink/cli
   # or
   brew install mldotink/tap/ink
   ```
3. Set the environment variable:
   ```bash
   export INK_API_KEY=dk_live_your_key_here
   ```

Or run `ink login` after installing the CLI.

## Skill surface

This repo ships one installable skill:

- [`ink`](skills/ink/SKILL.md) — Deploy apps, manage services, databases, DNS, custom domains, and workspaces on Ink

The skill uses the [Ink CLI](https://github.com/mldotink/cli) (`@mldotink/cli`). If you use the hosted installer script, it now prints CLI install commands when `ink` is missing.

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
