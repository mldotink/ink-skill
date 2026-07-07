---
name: ink
description: Deploy and manage cloud services on Ink (ml.ink): create projects, deploy services, deploy templates (databases, caches), manage DNS and custom domains, configure workspaces, and monitor deployments. Use this skill whenever the user mentions Ink, ml.ink, deployments, services, databases, templates, or cloud infrastructure on Ink, even if they don't say "Ink" explicitly.
allowed-tools: Bash(ink:*), Bash(which:*), Bash(command:*), Bash(npm:*), Bash(npx:*), Bash(brew:*), Bash(git:*), Write
---

# Use Ink

[Ink](https://ml.ink) is a cloud platform designed for AI agents to deploy and manage services autonomously. It makes deployments simple enough that fully autonomous agents can handle the entire lifecycle: create, deploy, monitor, and scale services without human intervention.

## Jaz / MCP-first rule

When running inside Jaz and Ink MCP tools are available, use the MCP tools first.
The container may have the `ink` CLI installed without CLI auth, so a failed
`ink whoami` is not a blocker when MCP auth is present. Use the CLI path only
when MCP tools are unavailable or the user explicitly asks for CLI commands.

## App Directory Shape

When creating a new app to deploy, make a dedicated directory for that app and
initialize git inside it. Do not treat the workspace root as the app repository.

```bash
mkdir -p apps/my-app
cd apps/my-app
git init
```

Keep one deployable Ink service per app directory unless the user asks for a
monorepo. Commit the app files before pushing to an Ink repo or deploying.

## CLI Preflight

When using the CLI path, verify the CLI is installed and authenticated:

```bash
command -v ink                    # CLI installed
ink whoami --json                 # authenticated; shows configured workspace/project
```

If the CLI is missing, install it:

```bash
npm install -g @mldotink/cli      # npm (macOS, Linux, Windows)
brew install mldotink/tap/ink     # Homebrew (macOS)
```

If not authenticated, run `ink login`.

## Configuration

Ink CLI resolves context in this order (highest priority first):

1. **CLI flags** -- `--api-key`, `--workspace`, `--project`, `--api-url`, `--oauth-url`, `--web-url`
2. **Environment** -- `INK_API_KEY`, `INK_WORKSPACE`, `INK_PROJECT`, `INK_API_URL`, `INK_OAUTH_URL`, `INK_WEB_URL`
3. **Local config** -- `.ink` file in current directory
4. **Global config** -- `~/.config/ink/config`

Use `ink whoami --json` to check current auth and inspect configured workspace/project. Treat that config as a hint, not proof it is valid: before deploying, run a scoped read such as `ink projects list --workspace <workspace> --json` or `ink services --workspace <workspace> --project <project> --json`. Prefer explicit `--workspace` and `--project` flags in automation when the target is known. Use `--json` on any command for machine-readable output.

## Git: Two Options

Ink supports two git providers. Use `ink whoami` to see which are available.

### Option 1: Ink Internal Git (default)

Zero-setup, works for everyone. No GitHub account needed.

```bash
ink repos create my-app           # creates repo, shows git remote URL
git remote add ink <url>          # add the remote
git push ink main                 # push code -- auto-triggers deployment
```

### Option 2: GitHub

Requires GitHub OAuth and GitHub App connected at https://ml.ink (Settings > GitHub).

```bash
ink deploy my-app --repo username/repo-name --host github --port 3000
```

### Auto-Redeploy

Both git providers trigger automatic redeployment on push. After pushing code, just poll `ink status <name>` to track progress -- you do not need to redeploy manually.

## Secrets & Environment Variables

Use `ink secrets` to manage env vars on running services. Changes are merged server-side and trigger an automatic redeploy.

**For sensitive values** (API keys, tokens, credentials), use `ink secrets import` to avoid leaking to shell history:

```bash
# From file
ink secrets import my-app --file .env

# From stdin (preferred for agents)
cat > .env.secrets <<EOF
DATABASE_URL=libsql://my-db-myworkspace.turso.io
DATABASE_AUTH_TOKEN=eyJhbG...
API_KEY=sk_live_xxx
EOF
ink secrets import my-app --file .env.secrets
rm .env.secrets
```

**For non-sensitive values**, `ink secrets set` is fine:

```bash
ink secrets set my-app NODE_ENV=production LOG_LEVEL=info
```

**Other operations:**

```bash
ink secrets list my-app                          # list current vars
ink secrets unset my-app OLD_KEY                 # remove a single var
ink secrets delete my-app KEY1 KEY2              # remove multiple vars
ink secrets import my-app --file .env --replace  # replace ALL vars (removes unspecified)
```

For initial deploy, use `--env-file` to pass secrets:

```bash
ink deploy my-app --repo my-app --port 3000 --env-file .env
```

Use `--env` only for non-sensitive values like `NODE_ENV=production`.

## Common Operations

```bash
ink services                                      # list all services
ink status my-app                                 # service details
ink logs my-app                                   # runtime logs
ink logs my-app --build                           # build logs
ink deploy my-app --repo my-app --port 3000       # deploy new service
ink deploy my-app --repo my-app --auth-policy public  # explicit public HTTP auth policy
ink redeploy my-app                               # redeploy existing
ink redeploy my-app --memory 1Gi --vcpu 1         # redeploy with new config
ink redeploy my-app --auth-policy org_sso         # require workspace SSO on Enterprise/self-hosted installs
ink delete my-app                                 # delete service

ink template                                      # list available templates
ink template info postgres                        # preview variables & services
ink template deploy postgres --name my-pg         # deploy a template stack

ink secrets set my-app KEY=value                  # set env vars (merges)
ink secrets import my-app --file .env             # import from file
ink secrets list my-app                           # list env vars
ink secrets unset my-app KEY                      # remove env var
ink secrets delete my-app KEY1 KEY2               # remove multiple

ink domains add my-app app.example.com            # add custom domain
ink domains remove my-app app.example.com         # remove custom domain

ink dns zones                                     # list DNS zones
ink dns records example.com                       # list records
ink dns add example.com --name sub --type A --content 1.2.3.4
ink dns delete example.com <record-id>

ink repos create my-app                           # create internal git repo
ink repos token my-app                            # get push token

ink projects list                                 # list projects
ink workspaces                                    # list workspaces
```

## Templates

Templates deploy pre-configured stacks (databases, caches, etc.) with a single command. The most common template is **postgres** — use it whenever the user needs a database.

**Always preview before deploying** to see required variables:

```bash
ink template                                      # list all templates
ink template info postgres                        # show variables, services, and example commands
ink template deploy postgres --name my-pg         # deploy (prompts for missing required vars)
ink template deploy postgres --name my-pg --var db_name=myapp  # pass variables inline
```

Use `--json` to get machine-readable output with connection credentials from `outputs`.

## Auth Policy

Ink Cloud public HTTP endpoints currently default to `public`; omit
`--auth-policy` for normal Ink Cloud deploys. Use `--auth-policy public` only
when being explicit.

Enterprise/self-hosted installs may enable route auth and a different default.
Use `--auth-policy org_sso` for signed-in workspace members or
`--auth-policy deployer_sso` for only the signed-in service creator. These SSO
policies require route auth and a configured SSO mode; they are not useful on
Ink Cloud while route auth is disabled.

When using MCP directly, prefer the top-level `auth_policy` field on
`service_create`/`service_update` rather than rebuilding the full `ports` array
only to change public HTTP auth.

## Deployment Flows

### Deploy a service

```bash
# 1. Create a repo and push code
ink repos create my-app
git remote add ink <gitRemote_from_output>
git push ink main

# 2. Deploy
ink deploy my-app --repo my-app --port 3000

# 3. Check status (status goes: queued -> building -> deploying -> active)
ink status my-app
```

### Deploy a full-stack app (API + frontend)

```bash
# 1. Deploy backend
ink repos create my-api
git remote add ink <url>
git push ink main
ink deploy my-api --repo my-api --port 8080

# 2. Wait for backend to be active and copy the returned endpoint URL
ink status my-api

# 3. Deploy frontend with backend URL
ink repos create my-frontend
git remote add ink-frontend <url>
git push ink-frontend main
ink deploy my-frontend --repo my-frontend --port 3000 \
  --env VITE_API_URL=<backend-url-from-ink-status>
```

### Deploy with a database

Use templates to provision databases. Always run `ink template info <slug>` first to see required variables.

```bash
# 1. Preview the template to see variables and services
ink template info postgres --json

# 2. Deploy the database template (returns connection credentials in outputs)
ink template deploy postgres --name my-pg --json
# Save the outputs (DATABASE_URL, etc.) from the response

# 3. Deploy your service
ink deploy my-app --repo my-app --port 3000

# 4. Wire credentials to your service
cat > .env.secrets <<EOF
DATABASE_URL=<DATABASE_URL from step 2>
EOF
ink secrets import my-app --file .env.secrets
rm .env.secrets
```

You can pass template variables with `--var KEY=VALUE`:

```bash
ink template deploy postgres --name my-pg --var db_name=myapp --var storage_gi=20
```

### Deploy a static site or SPA

For static files already present in the repo, use the `static` buildpack. No
local Python server or custom start command is needed. No `--port` is needed in
the CLI path because Ink serves the files via nginx automatically.

```bash
ink repos create my-site
git remote add ink <url>
git push ink main
ink deploy my-site --repo my-site --buildpack static
```

For prebuilt static files in a subdirectory, use `static` with `--publish-dir`:

```bash
ink deploy my-site --repo my-site --buildpack static --publish-dir dist
```

When using MCP directly for static sites, prefer `build_pack: "static"` plus
`publish_directory` when needed. Omit raw `ports` unless you are intentionally
changing visibility/auth behavior; if you do pass `ports`, provide the complete
port object, including `port`, `protocol`, `visibility`, and `auth_policy`.

For frontend apps (React, Vue, Vite, Next.js static export, etc.) that need Ink to run a build first, leave the buildpack as railpack and specify the build output directory:

```bash
ink repos create my-site
git remote add ink <url>
git push ink main
ink deploy my-site --repo my-site --publish-dir dist
```

Use `--publish-dir` to specify where the built or prebuilt files are (`dist`, `build`, `out`, etc.).

For SPAs with an API backend, pass the API URL as a build-time env var:

```bash
ink deploy my-site --repo my-site --publish-dir dist \
  --env VITE_API_URL=<backend-url-from-ink-status>
```

### Deploy from a monorepo

```bash
# 1. Create one repo for the monorepo
ink repos create my-monorepo
git remote add ink <url>
git push ink main

# 2. Deploy backend from backend/ subdirectory
ink deploy mono-api --repo my-monorepo --root-dir backend --port 8080

# 3. Deploy frontend from frontend/ subdirectory
ink deploy mono-web --repo my-monorepo --root-dir frontend --publish-dir dist \
  --env VITE_API_URL=<backend-url-from-ink-status>
```

### Deploy from GitHub

Requires GitHub OAuth and App connected at https://ml.ink.

```bash
ink deploy my-app --repo username/repo-name --host github --port 3000
```

Pushes to the GitHub repo automatically trigger redeployment via webhook.

### Add a custom domain

Requires a DNS zone delegated to Ink first at https://ml.ink/dns.

```bash
ink dns zones                                     # verify zone is active
ink domains add my-app app.example.com            # auto-creates DNS + TLS
```

### Update a service

Use `ink secrets` for env var changes. Use `ink redeploy` for resource/config
changes. Pushing code auto-redeploys, so you do not need to call redeploy after
a normal git push.

```bash
# Add or update non-sensitive env vars
ink secrets set my-app NODE_ENV=production LOG_LEVEL=info

# Add secrets via file (avoids shell history exposure)
ink secrets import my-app --file .env.secrets

# Scale up memory and CPU
ink redeploy my-app --memory 1Gi --vcpu 1
```

### Debug a failing deployment

```bash
ink status my-app                                 # check status and error
ink logs my-app                                   # check runtime logs
ink logs my-app --build                           # check build logs
```

## Service Configuration

| Option | Values | Default |
|--------|--------|---------|
| Memory | 128Mi, 256Mi, 512Mi, 1024Mi, 2048Mi, 4096Mi | 256Mi |
| vCPUs | 0.1, 0.2, 0.25, 0.3, 0.4, 0.5, 1, 2, 3, 4 | 0.25 |
| Region | Active platform regions; Ink Cloud is currently `us-east-1` | omit for platform default |
| Branch | any git branch | main |

## Guidelines

- Install the CLI first. If `command -v ink` fails, install with `npm install -g @mldotink/cli`.
- Start with `ink whoami --json` when using the CLI path. Confirm auth and inspect configured workspace/project before using mutating commands.
- Validate context before deploying. Use `ink projects list --workspace <workspace> --json` or `ink services --workspace <workspace> --project <project> --json`; stale local/global config can point at a workspace or project that no longer exists.
- Check `ink services` before deploying to see if a service already exists. Use `ink deploy` for new services and `ink redeploy` for existing ones.
- Pushing code auto-redeploys. After `git push`, just poll `ink status` to track progress.
- Use `--json` for machine-readable output when you need to parse results.
- Confirm the repo URL and branch with the user before deploying from GitHub or from an ambiguous repo.
- Use `ink secrets import` for sensitive values. Write to a temp file, import, delete the file. Never pass secrets as CLI arguments.
- Use `ink secrets set` only for non-sensitive vars like `NODE_ENV=production`.
- Never use `ink redeploy --env` to update vars unless you intentionally want to replace service env vars through a redeploy; prefer `ink secrets`.
- Never hardcode or guess secret values. Secrets should come from the user, template deploy outputs, or other Ink CLI output.
- Show the service URL after successful deployment. Do not construct or guess managed app URL formats; use the endpoint returned by Ink.
- Zone delegation for custom domains must be set up by the user at https://ml.ink/dns before using `ink domains add`.
- Track what you deploy. After creating repos, services, or templates, record the workspace, project, service names, and endpoints in the project's `CLAUDE.md` or `AGENTS.md`.

Example deployment note:

```markdown
## Ink Deployment
- Workspace: my-team
- Project: backend
- Services: my-api (<url-from-ink-status>), my-worker
- Git remote: ink (git.ml.ink/my-team/my-api)
```
