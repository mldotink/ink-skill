---
name: ink
description: Deploy and manage cloud services on Ink (ml.ink) using the GraphQL API. Use when deploying apps, managing services, databases, DNS, custom domains, or checking infrastructure status on Ink.
license: MIT
compatibility: Requires curl, jq, git, and access to the internet. Designed for Claude Code.
metadata:
  author: mldotink
  version: "1.0"
---

# Ink Cloud Deployment Skill

Deploy and manage cloud services on [Ink](https://ml.ink) using the GraphQL API.

## Setup

Set the `INK_API_KEY` environment variable with your Ink API key (starts with `dk_live_`).
Get one from the [Ink dashboard](https://ml.ink) under Settings > API Keys.

## API

**Endpoint:** `https://api.ml.ink/graphql`
**Auth:** `Authorization: Bearer $INK_API_KEY`
**Method:** POST with `{"query": "...", "variables": {...}}`

Use `curl -s` and pipe through `jq` for readable output.

## First Step: Fetch the Schema

Before making any API calls, always fetch the latest schema. It contains all queries, mutations, types, input fields with descriptions, and default values:

```bash
curl -s https://api.ml.ink/schema
```

Read the schema output to understand the exact fields, arguments, and types available. The schema is self-documenting with descriptions on every field. Use it as your primary reference rather than memorizing queries.

## Deployment Flows

### Deploy a full-stack app (API backend + frontend)

Deploy a backend API, then a frontend that connects to it via env var.

```bash
# 1. Check existing services to avoid duplicates
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ serviceList { nodes { name status fqdn } } }"}' | jq

# 2. Create a repo for the backend and push code
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "mutation($input: RepoCreateInput!) { repoCreate(input: $input) { repo gitRemote } }", "variables": {"input": {"name": "my-api"}}}' | jq

# Push code using the returned gitRemote URL
git remote add ink <gitRemote_from_above>
git push ink main

# 3. Deploy the backend API
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "mutation($input: CreateServiceInput!) { serviceCreate(input: $input) { serviceId name status } }", "variables": {"input": {"name": "my-api", "repo": "my-api", "port": 8080, "memory": "512Mi", "envVars": [{"key": "NODE_ENV", "value": "production"}]}}}' | jq

# 4. Poll until backend is active (status goes: queued -> building -> deploying -> active)
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "query($id: ID!) { serviceGet(id: $id) { status fqdn errorMessage } }", "variables": {"id": "SERVICE_ID_FROM_STEP_3"}}' | jq

# 5. Create a repo for the frontend and push code
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "mutation($input: RepoCreateInput!) { repoCreate(input: $input) { repo gitRemote } }", "variables": {"input": {"name": "my-frontend"}}}' | jq

git remote add ink-frontend <gitRemote_from_above>
git push ink-frontend main

# 6. Deploy the frontend with the backend URL injected as env var
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "mutation($input: CreateServiceInput!) { serviceCreate(input: $input) { serviceId name status } }", "variables": {"input": {"name": "my-frontend", "repo": "my-frontend", "port": 3000, "envVars": [{"key": "VITE_API_URL", "value": "https://my-api.ml.ink"}]}}}' | jq
```

Result: Backend at `https://my-api.ml.ink`, frontend at `https://my-frontend.ml.ink` with the API URL baked in.

### Deploy an app with a database

Provision a SQLite database, then deploy a service wired to it.

```bash
# 1. Create a database — returns connection credentials
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "mutation($input: CreateResourceInput!) { resourceCreate(input: $input) { name databaseUrl authToken } }", "variables": {"input": {"name": "my-db"}}}' | jq

# Save the databaseUrl and authToken from the response

# 2. Create repo and push code
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "mutation($input: RepoCreateInput!) { repoCreate(input: $input) { repo gitRemote } }", "variables": {"input": {"name": "my-app"}}}' | jq

git remote add ink <gitRemote>
git push ink main

# 3. Deploy the service with database credentials as env vars
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "mutation($input: CreateServiceInput!) { serviceCreate(input: $input) { serviceId name status } }", "variables": {"input": {"name": "my-app", "repo": "my-app", "port": 3000, "envVars": [{"key": "DATABASE_URL", "value": "<databaseUrl_from_step_1>"}, {"key": "DATABASE_AUTH_TOKEN", "value": "<authToken_from_step_1>"}]}}}' | jq
```

### Deploy from a monorepo

Deploy multiple services from different subdirectories of the same repo.

```bash
# 1. Create one repo for the monorepo
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "mutation($input: RepoCreateInput!) { repoCreate(input: $input) { repo gitRemote } }", "variables": {"input": {"name": "my-monorepo"}}}' | jq

git remote add ink <gitRemote>
git push ink main

# 2. Deploy the backend from the backend/ subdirectory
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "mutation($input: CreateServiceInput!) { serviceCreate(input: $input) { serviceId name status } }", "variables": {"input": {"name": "mono-api", "repo": "my-monorepo", "rootDirectory": "backend", "port": 8080}}}' | jq

# 3. Deploy the frontend from the frontend/ subdirectory as a static SPA
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "mutation($input: CreateServiceInput!) { serviceCreate(input: $input) { serviceId name status } }", "variables": {"input": {"name": "mono-web", "repo": "my-monorepo", "rootDirectory": "frontend", "publishDirectory": "dist", "envVars": [{"key": "VITE_API_URL", "value": "https://mono-api.ml.ink"}]}}}' | jq
```

The `rootDirectory` sets the build context. The `publishDirectory` tells railpack to build the app then serve the output as static files via nginx.

### Deploy from GitHub

Use GitHub repos instead of Ink's internal git. Requires the Ink GitHub App installed.

```bash
# Deploy directly from a GitHub repo (public repos work without GitHub App)
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "mutation($input: CreateServiceInput!) { serviceCreate(input: $input) { serviceId name status } }", "variables": {"input": {"name": "my-app", "repo": "username/repo-name", "host": "github", "port": 3000}}}' | jq
```

With the GitHub App installed, pushes to the repo automatically trigger redeployment.

### Add a custom domain

Requires a DNS zone delegated to Ink first (done via the web dashboard at https://ml.ink/dns).

```bash
# 1. Check the zone is active
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ dnsListZones { zone status } }"}' | jq

# 2. Attach the domain to a service (auto-creates DNS records and TLS cert)
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "mutation($name: String!, $domain: String!) { domainAdd(name: $name, domain: $domain) { domain status message } }", "variables": {"name": "my-app", "domain": "app.example.com"}}' | jq
```

### Redeploy or update a service

```bash
# Redeploy with no changes (pulls latest code from the repo)
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "mutation($input: UpdateServiceInput!) { serviceUpdate(input: $input) { serviceId status } }", "variables": {"input": {"name": "my-app"}}}' | jq

# Scale up memory and CPU
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "mutation($input: UpdateServiceInput!) { serviceUpdate(input: $input) { serviceId status } }", "variables": {"input": {"name": "my-app", "memory": "1Gi", "vcpus": "1"}}}' | jq

# Update env vars (replaces all existing vars)
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "mutation($input: UpdateServiceInput!) { serviceUpdate(input: $input) { serviceId status } }", "variables": {"input": {"name": "my-app", "envVars": [{"key": "NODE_ENV", "value": "production"}, {"key": "API_KEY", "value": "sk-xxx"}]}}}' | jq
```

### Debug a failing deployment

```bash
# Check status and error message
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "query($id: ID!) { serviceGet(id: $id) { name status errorMessage fqdn } }", "variables": {"id": "SERVICE_ID"}}' | jq

# Check the action log for recent operations
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ actionLogList(limit: 10) { nodes { action entityType entityId source createdAt } } }"}' | jq
```

For build and runtime logs, use the MCP tool `service_get` with `deploy_log_lines` and `runtime_log_lines` parameters -- logs are not yet available via the GraphQL API.

## Quick Reference

These are the most common individual operations. For full field details, check the schema (`curl -s https://api.ml.ink/schema`).

| Operation | Query/Mutation |
|---|---|
| Who am I | `{ accountStatus { id email hasGitHubApp defaultWorkspace } }` |
| List services | `{ serviceList { nodes { name status fqdn } totalCount } }` |
| Get service | `serviceGet(id: ID!) { ... }` |
| Create service | `serviceCreate(input: CreateServiceInput!) { serviceId name status }` |
| Update/redeploy | `serviceUpdate(input: UpdateServiceInput!) { serviceId status }` |
| Delete service | `serviceDelete(name: "...") { message }` |
| List projects | `{ projectList { nodes { name slug } } }` |
| Delete project | `projectDelete(slug: "...") ` |
| Create database | `resourceCreate(input: { name: "..." }) { databaseUrl authToken }` |
| List databases | `{ resourceList { nodes { name status } } }` |
| Get database | `resourceGet(id: ID!) { ... }` |
| Delete database | `resourceDelete(name: "...") { message }` |
| Create repo | `repoCreate(input: { name: "..." }) { repo gitRemote }` |
| Get push token | `repoGetToken(input: { name: "..." }) { gitRemote expiresAt }` |
| List DNS zones | `{ dnsListZones { zone status } }` |
| List DNS records | `dnsListRecords(zone: "...") { name type content }` |
| Add DNS record | `dnsAddRecord(zone: "...", name: "...", type: "...", content: "...") { id }` |
| Delete DNS record | `dnsDeleteRecord(zone: "...", recordId: ID!)` |
| Add custom domain | `domainAdd(name: "svc", domain: "app.example.com") { status }` |
| Remove custom domain | `domainRemove(name: "svc") { message }` |
| List workspaces | `{ workspaceList { name slug role } }` |
| Create workspace | `workspaceCreate(name: "...", slug: "...") { id }` |
| Send chat | `chatSend(workspaceSlug: "...", content: "...") { seq }` |
| Read chat | `chatRead(workspaceSlug: "...") { messages { senderName content } }` |
| Action log | `{ actionLogList(limit: 20) { nodes { action entityType createdAt } } }` |

## Guidelines

- **Always fetch the schema first** with `curl -s https://api.ml.ink/schema` before making API calls. It has the latest fields, types, and defaults.
- **Check serviceList before deploying** to see if a service already exists. Use `serviceCreate` for new services and `serviceUpdate` to modify or redeploy existing ones.
- **Poll serviceGet after create/update** to track deployment progress. Status transitions: queued -> building -> deploying -> active (or failed).
- **Env vars on update replace all existing vars.** Include all vars you want to keep, not just the new ones.
- **Internal git is the default** (`host: "ink"`). No GitHub setup needed. Create a repo, push code, deploy.
- **Service-to-service calls** use the `internalUrl` (e.g. `http://svc.ns.svc.cluster.local:port`) instead of the public FQDN. Lower latency, no TLS overhead.
- **Memory:** 256Mi (default, fine for most apps), 512Mi, 1Gi, 2Gi, 4Gi, 8Gi.
- **vCPUs:** 0.25 (default), 0.5, 1, 2, 4.
- When deploying, confirm the repo URL and branch with the user first.
- For environment variables containing secrets, ask the user rather than guessing values.
- Show the service URL (`fqdn`) after successful deployment.
- Zone delegation (for custom domains) must be set up by the user at https://ml.ink/dns before you can use `domainAdd`.
