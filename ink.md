# Ink Cloud Deployment Skill

Deploy and manage cloud services on [Ink](https://ml.ink) using the GraphQL API.

## Setup

Set the `INK_API_KEY` environment variable with your Ink API key (starts with `dk_live_`).
Get one from the [Ink dashboard](https://ml.ink) under Settings > API Keys.

## API

**Endpoint:** `https://api.ml.ink/graphql`
**Auth:** `Authorization: Bearer $INK_API_KEY`
**Method:** POST with `{"query": "...", "variables": {...}}`

Use `curl` to call the API. Always use `-s` and pipe through `jq` for readable output.

## Discover the API

Before making calls, introspect the endpoint to see all available queries, mutations, types, and input types. Introspection does not require authentication:

```bash
curl -s https://api.ml.ink/graphql \
  -H "Content-Type: application/json" \
  -d '{"query": "{ __schema { queryType { name } mutationType { name } types { kind name fields(includeDeprecated: false) { name args { name type { kind name ofType { kind name ofType { kind name ofType { kind name } } } } } type { kind name ofType { kind name ofType { kind name ofType { kind name } } } } } inputFields { name type { kind name ofType { kind name ofType { kind name ofType { kind name } } } } } enumValues { name } } } }"}' | jq
```

This returns the full schema -- all queries, mutations, return types, input types (like `CreateServiceInput`), and enums. Use it to discover fields, arguments, and types before constructing queries.

An example response is saved in `example-schema.json` in this directory.

## Common Operations

### Account info

```bash
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ accountStatus { id email displayName username hasGitHubApp defaultWorkspace } }"}' | jq
```

### Services

```bash
# List services
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ serviceList { nodes { id name status fqdn memory vcpus project { slug } } totalCount } }"}' | jq

# Get service details
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "query($id: ID!) { serviceGet(id: $id) { id name status repo branch fqdn port memory vcpus customDomain customDomainStatus envVars { key value } commitHash createdAt updatedAt } }", "variables": {"id": "SERVICE_ID"}}' | jq

# Create a service
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "mutation($input: CreateServiceInput!) { serviceCreate(input: $input) { serviceId name status repo internalUrl } }", "variables": {"input": {"name": "my-service", "repo": "https://github.com/user/repo", "branch": "main", "port": 3000, "memory": "512Mi", "vcpus": "0.5", "envVars": [{"key": "NODE_ENV", "value": "production"}]}}}' | jq

# Update a service (redeploy, change config, env vars, etc.)
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "mutation($input: UpdateServiceInput!) { serviceUpdate(input: $input) { serviceId name status } }", "variables": {"input": {"name": "my-service", "repo": "https://github.com/user/repo", "branch": "main", "port": 3000, "memory": "512Mi", "vcpus": "0.5", "envVars": [{"key": "NODE_ENV", "value": "production"}]}}}' | jq

# Delete a service
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "mutation($name: String!) { serviceDelete(name: $name) { serviceId name message } }", "variables": {"name": "my-service"}}' | jq
```

CreateServiceInput / UpdateServiceInput fields:
- `name` (required): service name (becomes `name.ml.ink`)
- `repo`: git repository URL (required for create)
- `project`: project slug (optional, uses default)
- `workspaceSlug`: workspace (optional, uses default)
- `host`: git host override (e.g. "ml.ink" for internal git)
- `branch`: git branch (default: "main")
- `port`: container port
- `envVars`: list of `{key, value}` pairs
- `buildPack`: build pack type
- `memory`: memory limit ("256Mi", "512Mi", "1Gi", "2Gi", "4Gi", "8Gi")
- `vcpus`: CPU allocation ("0.25", "0.5", "1", "2", "4")
- `buildCommand`: custom build command
- `startCommand`: custom start command
- `publishDirectory`: for static sites
- `rootDirectory`: monorepo subdirectory
- `dockerfilePath`: path to Dockerfile
- `regions`: deployment regions (create only)

### Projects

```bash
# List projects
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ projectList { nodes { id name slug services { name status } } totalCount } }"}' | jq

# Delete project
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "mutation($slug: String!) { projectDelete(slug: $slug) }", "variables": {"slug": "my-project"}}' | jq
```

### Resources (Databases)

```bash
# List resources
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ resourceList { nodes { id name type provider region status metadata { size hostname } project { slug } } totalCount } }"}' | jq

# Create a database
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "mutation($input: CreateResourceInput!) { resourceCreate(input: $input) { resourceId name type region databaseUrl authToken status } }", "variables": {"input": {"name": "my-db"}}}' | jq

# Get resource details
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "query($id: ID!) { resourceGet(id: $id) { id name type provider region status metadata { size hostname } createdAt } }", "variables": {"id": "RESOURCE_ID"}}' | jq

# Delete a resource
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "mutation($name: String!) { resourceDelete(name: $name) { resourceId name message } }", "variables": {"name": "my-db"}}' | jq
```

### Git Repos (Internal Git)

Push code directly to Ink's internal git (no GitHub required).

```bash
# Create a repo
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "mutation($input: RepoCreateInput!) { repoCreate(input: $input) { repo gitRemote expiresAt message } }", "variables": {"input": {"name": "my-app"}}}' | jq

# Get a push token (for existing repos)
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "mutation($input: RepoGetTokenInput!) { repoGetToken(input: $input) { gitRemote expiresAt } }", "variables": {"input": {"name": "my-app"}}}' | jq
```

After creating a repo, push code using the returned `gitRemote` URL:

```bash
git remote add ink <gitRemote>
git push ink main
```

Then create a service pointing to the internal repo:

```bash
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "mutation($input: CreateServiceInput!) { serviceCreate(input: $input) { serviceId name status internalUrl } }", "variables": {"input": {"name": "my-app", "repo": "my-app", "host": "ml.ink", "port": 3000}}}' | jq
```

### DNS

```bash
# List zones
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ dnsListZones { id zone status records { id name type content ttl } } }"}' | jq

# List records in a zone
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "query($zone: String!) { dnsListRecords(zone: $zone) { id name type content ttl managed } }", "variables": {"zone": "example.com"}}' | jq

# Add a DNS record
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "mutation($zone: String!, $name: String!, $type: String!, $content: String!) { dnsAddRecord(zone: $zone, name: $name, type: $type, content: $content) { id name type content ttl } }", "variables": {"zone": "example.com", "name": "www", "type": "CNAME", "content": "my-app.ml.ink"}}' | jq

# Delete a DNS record
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "mutation($zone: String!, $recordId: ID!) { dnsDeleteRecord(zone: $zone, recordId: $recordId) }", "variables": {"zone": "example.com", "recordId": "RECORD_ID"}}' | jq
```

Note: Zone creation/verification/deletion must be done through the web dashboard at https://ml.ink/dns.

### Custom Domains

Attach a custom domain to a service (requires a verified DNS zone first):

```bash
# Add custom domain
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "mutation($name: String!, $domain: String!) { domainAdd(name: $name, domain: $domain) { serviceId domain status message } }", "variables": {"name": "my-service", "domain": "app.example.com"}}' | jq

# Remove custom domain
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "mutation($name: String!) { domainRemove(name: $name) { serviceId message } }", "variables": {"name": "my-service"}}' | jq
```

### Workspaces

```bash
# List workspaces
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ workspaceList { id name slug isDefault role } }"}' | jq

# Create workspace
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "mutation($name: String!, $slug: String!) { workspaceCreate(name: $name, slug: $slug) { id name slug } }", "variables": {"name": "My Team", "slug": "my-team"}}' | jq

# List members
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "query($ws: String!) { workspaceListMembers(workspaceSlug: $ws) { id email displayName role joinedAt } }", "variables": {"ws": "my-team"}}' | jq

# List invites for a workspace
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "query($ws: String!) { workspaceListInvites(workspaceSlug: $ws) { id role status inviteeDisplayName createdAt } }", "variables": {"ws": "my-team"}}' | jq

# Invite a user
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "mutation($wsId: ID!, $user: String!, $role: String) { workspaceInvite(workspaceId: $wsId, user: $user, role: $role) { id role status } }", "variables": {"wsId": "WORKSPACE_ID", "user": "user@example.com", "role": "member"}}' | jq

# Accept/decline invites
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "mutation($id: ID!) { workspaceAcceptInvite(inviteId: $id) }", "variables": {"id": "INVITE_ID"}}' | jq

# Revoke an invite (admin/owner)
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "mutation($id: ID!) { workspaceRevokeInvite(inviteId: $id) }", "variables": {"id": "INVITE_ID"}}' | jq

# Remove a member (admin/owner)
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "mutation($wsId: ID!, $userId: ID!) { workspaceRemoveMember(workspaceId: $wsId, userId: $userId) }", "variables": {"wsId": "WORKSPACE_ID", "userId": "USER_ID"}}' | jq

# Delete workspace
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "mutation($id: ID!) { workspaceDelete(id: $id) }", "variables": {"id": "WORKSPACE_ID"}}' | jq
```

### Action Log

```bash
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ actionLogList(limit: 20) { nodes { id action entityType entityId source createdAt } } }"}' | jq
```

### Chat

```bash
# Send a message
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "mutation($ws: String!, $content: String!) { chatSend(workspaceSlug: $ws, content: $content) { seq messageId } }", "variables": {"ws": "default", "content": "Hello team!"}}' | jq

# Read messages
curl -s https://api.ml.ink/graphql \
  -H "Authorization: Bearer $INK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query": "query($ws: String!) { chatRead(workspaceSlug: $ws) { messages { seq senderName content createdAt } nextCursor hasMore } }", "variables": {"ws": "default"}}' | jq
```

## Guidelines

- Always check `serviceList` before deploying to see if a service already exists. Use `serviceCreate` for new services and `serviceUpdate` to modify existing ones.
- Use `workspaceSlug` parameter when the user has multiple workspaces.
- Memory values: "256Mi", "512Mi", "1Gi", "2Gi", "4Gi", "8Gi".
- vCPU values: "0.25", "0.5", "1", "2", "4".
- When deploying, always confirm the repo URL and branch with the user first.
- For environment variables, ask the user rather than guessing values.
- Show the service URL (fqdn) after successful deployment.
- Zone creation/verification/deletion is done through the web dashboard at https://ml.ink/dns. The API supports listing zones, managing records, and attaching custom domains.
- Use introspection to discover input types and return fields if you need more detail than shown here.
