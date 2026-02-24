---
name: cvm2-agent-api
description: Build and test flows programmatically via the Agent API
version: 1.0.0
---

# CVM2 Agent API

Build, test, and execute flows programmatically. The Agent API provides isolated sessions with dedicated plugin branches for safe experimentation.

## Overview

The Agent API is a REST API for LLM agents to:
- Create isolated flow development sessions
- Build flows by adding nodes and edges
- Execute and monitor flow runs
- Test plugin methods directly

**Base URL:** `https://<api-host>/api/v1/agent`

**Authentication:** All endpoints require API key:
```
Authorization: Bearer <api_key>
```

## Workflow

```
Agent API Flow Development Checklist
 - [ ] Start a session (new or edit existing)
 - [ ] Discover available node types
 - [ ] Build the flow (nodes + edges)
 - [ ] Insert test data
 - [ ] Run and monitor
 - [ ] Inspect results
 - [ ] Commit & submit for review
```

### Step 1: Start a Session

There are two ways to start a session:

#### Option A: Create a NEW Flow

Creates a new empty flow, agent version, and dedicated plugin branch:

```bash
curl -X POST "$BASE_URL/api/v1/agent/start/new" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "My Test Flow",
    "description": "Testing SMS campaign",
    "agent_id": "claude-session-001",
    "reason": "Building SMS notification flow",
    "ttl_minutes": 60
  }'
```

Response:
```json
{
  "flow_id": "abc-123",
  "version_id": "agent-claude-session-001-...",
  "view_url": "https://example.com/workflow/abc-123/agent-claude-session-001-...",
  "branch": {
    "name": "agent/abc-123",
    "source": "dev"
  },
  "_note": "**Action Required:** Share this link with the user...\n- **View URL:** https://..."
}
```

#### Option B: EDIT an Existing Flow

First, discover existing flows:

```bash
# Search for flows
curl "$BASE_URL/api/v1/agent/flows?search=sms&limit=10" \
  -H "Authorization: Bearer $TOKEN"

# Response includes version_summary for each flow:
{
  "flows": [{
    "flow_id": "abc-123",
    "name": "SMS Campaign",
    "version_summary": { "total": 3, "production": true, "active_agent_sessions": 0 }
  }],
  "total": 42, "limit": 10, "offset": 0
}
```

Get flow details with all versions:

```bash
curl "$BASE_URL/api/v1/agent/flows/$FLOW_ID" \
  -H "Authorization: Bearer $TOKEN"

# Response shows all versions with node/edge counts:
{
  "flow_id": "abc-123",
  "name": "SMS Campaign",
  "versions": [
    { "version_id": "production", "is_production": true, "node_count": 5, "edge_count": 4 },
    { "version_id": "agent-xyz-...", "version_type": "agent", "node_count": 6, "edge_count": 5 }
  ],
  "active_agent_sessions": 1
}
```

Fork from a version to edit:

```bash
curl -X POST "$BASE_URL/api/v1/agent/start/edit" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "flow_id": "abc-123",
    "source_version_id": "production",
    "agent_id": "claude-session-001",
    "reason": "Adding retry logic to SMS node",
    "ttl_minutes": 60
  }'
```

Response:
```json
{
  "flow_id": "abc-123",
  "flow_name": "SMS Campaign",
  "version_id": "agent-claude-session-001-...",
  "source_version_id": "production",
  "view_url": "https://example.com/workflow/abc-123/agent-claude-session-001-...",
  "forked_structure": {
    "node_count": 5,
    "edge_count": 4,
    "nodes_copied": ["ingress", "sms.sendSms", "wait", "junction", "end"]
  },
  "flow_summary": { "has_production_version": true, "active_agent_sessions": 1 },
  "_note": "**Forked from:** `production`\n**Action Required:** Share this link..."
}
```

#### Response Fields

**Important fields:**
- `flow_id`, `version_id` - Use in subsequent API calls
- `view_url` - **Share this with the user** so they can see the flow being built
- `branch.name` - **CRITICAL**: Use this branch for plugin development (see Plugin Development below)
- `_note` - Markdown-formatted instructions for the agent

**Optional parameters for `/start/new`:**
- `source_branch` - Override the source branch (e.g., `"source_branch": "master"`)
- `create_branch` - Set to `false` to skip plugin branch creation

### Step 2: Discover Node Types

Get available plugins and their methods:

```bash
curl "$BASE_URL/api/v1/agent/node-types/$BRANCH_NAME" \
  -H "Authorization: Bearer $TOKEN"
```

> **Note:** `BRANCH_NAME` defaults to `dev`. Use your session's `branch.name` if you need custom plugins from your agent branch (URL-encode slashes: `agent/abc-123` → `agent%2Fabc-123`).

Returns all node types with their settings schemas. Use the `id` field (e.g., `sms.sendSms`) as `node_type` when adding nodes.

### Step 3: Build the Flow

**Add an ingress node (entry point):**
```bash
curl -X POST "$BASE_URL/api/v1/agent/flows/$FLOW_ID/versions/$VERSION_ID/nodes" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "node_type": "ingress",
    "label": "Start"
  }'
```

**Add a plugin node:**
```bash
curl -X POST "$BASE_URL/api/v1/agent/flows/$FLOW_ID/versions/$VERSION_ID/nodes" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "node_type": "sms.sendSms",
    "label": "Send SMS",
    "config": {
      "message": "Hello <%= name %>"
    }
  }'
```

> **Note:** Template syntax varies by plugin. Most use EJS (`<%= variable %>`).

**Update an existing node:**

Use **PATCH** for partial updates (recommended):
```bash
curl -X PATCH "$BASE_URL/api/v1/agent/flows/$FLOW_ID/versions/$VERSION_ID/nodes/$NODE_ID" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "config": {
      "message": "Updated message: Hello <%= name %>!"
    }
  }'
```
PATCH merges with existing values — only the fields you provide are updated.

Use **PUT** for full replacement:
```bash
curl -X PUT "$BASE_URL/api/v1/agent/flows/$FLOW_ID/versions/$VERSION_ID/nodes/$NODE_ID" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "label": "Send SMS",
    "position": { "x": 100, "y": 200 },
    "config": { "message": "Hello <%= name %>" },
    "meta": {},
    "disabled": false
  }'
```
PUT replaces the entire node — omitted fields are reset to defaults.

The `$NODE_ID` can be the short form (`node-abc`) or full form (`node-abc:version-id`).

**Connect nodes with an edge:**
```bash
curl -X POST "$BASE_URL/api/v1/agent/flows/$FLOW_ID/versions/$VERSION_ID/edges" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "from_node": "node-ingress-id",
    "to_node": "node-sms-id"
  }'
```

**Validate the flow:**
```bash
curl "$BASE_URL/api/v1/agent/flows/$FLOW_ID/versions/$VERSION_ID/validate" \
  -H "Authorization: Bearer $TOKEN"
```

### Step 4: Insert Test Data

Insert a work item into the ingress node:

```bash
curl -X POST "$BASE_URL/api/v1/agent/flows/$FLOW_ID/versions/$VERSION_ID/insert" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "node_id": "node-ingress-id",
    "payload": {
      "phone": "+27821234567",
      "name": "John"
    }
  }'
```

### Step 5: Run and Monitor

**Run the flow (auto-tick until complete):**
```bash
curl -X POST "$BASE_URL/api/v1/agent/flows/$FLOW_ID/versions/$VERSION_ID/run" \
  -H "Authorization: Bearer $TOKEN"
```

**Or tick manually (one step at a time):**
```bash
curl -X POST "$BASE_URL/api/v1/agent/flows/$FLOW_ID/versions/$VERSION_ID/tick" \
  -H "Authorization: Bearer $TOKEN"
```

**Step-by-step node execution (2-step process):**

For precise control, you can insert data at a specific node and then tick just that node:

```bash
# Step 1: Insert a work item at a specific node
curl -X POST "$BASE_URL/api/v1/agent/flows/$FLOW_ID/versions/$VERSION_ID/insert" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "node_id": "node-abc",
    "payload": { "key": "value" }
  }'
# Returns: { "work_id": "uuid", "run_id": "uuid", ... }

# Step 2: Tick only that specific node
curl -X POST "$BASE_URL/api/v1/agent/flows/$FLOW_ID/versions/$VERSION_ID/tick" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "node_id": "node-abc"
  }'
```

This is useful for:
- Testing individual nodes in isolation
- Debugging flow behavior step-by-step
- Verifying node configuration before running the full flow

**Check status:**
```bash
curl "$BASE_URL/api/v1/agent/flows/$FLOW_ID/versions/$VERSION_ID/status" \
  -H "Authorization: Bearer $TOKEN"
```

### Step 6: Inspect Results

**View logs:**
```bash
curl "$BASE_URL/api/v1/agent/flows/$FLOW_ID/versions/$VERSION_ID/logs" \
  -H "Authorization: Bearer $TOKEN"
```

**Get work item trace:**
```bash
curl "$BASE_URL/api/v1/agent/work-items/$WORK_ID/trace" \
  -H "Authorization: Bearer $TOKEN"
```

**View items at a node:**
```bash
curl "$BASE_URL/api/v1/agent/flows/$FLOW_ID/versions/$VERSION_ID/nodes/$NODE_ID/items" \
  -H "Authorization: Bearer $TOKEN"
```

**Inspect a single node (config, connections, validation):**
```bash
curl "$BASE_URL/api/v1/agent/flows/$FLOW_ID/versions/$VERSION_ID/nodes/$NODE_ID" \
  -H "Authorization: Bearer $TOKEN"
```

Response:
```json
{
  "id": "node-abc:version-id",
  "type": "sms.sendSms",
  "label": "Send SMS",
  "config": { "message": "Hello <%= name %>" },
  "connections": {
    "incoming": [{ "edge_id": "edge-1", "from_node": "node-start" }],
    "outgoing": [{ "edge_id": "edge-2", "to_node": "node-end" }]
  },
  "validation": { "valid": true, "errors": [], "warnings": [] }
}
```

### Step 7: Commit & Submit for Review

When your flow is ready, commit your work to submit it for human review. This creates a "submission" (similar to a Pull Request) that a human can accept or reject.

**Commit your work:**
```bash
curl -X POST "$BASE_URL/api/v1/agent/versions/$VERSION_ID/commit" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "message": "Added SMS retry logic with exponential backoff",
    "title": "SMS Flow with Retry Support",
    "description": "Implements retry mechanism for failed SMS sends"
  }'
```

Response:
```json
{
  "version_id": "agent-claude-session-001-...",
  "flow_id": "abc-123",
  "submission": {
    "id": 42,
    "status": "pending",
    "title": "SMS Flow with Retry Support",
    "created_at": "2026-02-24T10:00:00Z",
    "updated_at": "2026-02-24T10:00:00Z",
    "commit_count": 1
  },
  "commit": {
    "id": 15,
    "message": "Added SMS retry logic with exponential backoff",
    "committed_at": "2026-02-24T10:00:00Z"
  },
  "view_url": "https://example.com/workflow/abc-123/agent-...",
  "_note": "Your work has been submitted for human review. You may continue editing - subsequent commits will update this submission."
}
```

**Key behaviors:**
- **First commit:** Creates a submission and removes the version TTL (your work is preserved indefinitely)
- **Subsequent commits:** Updates the existing submission and adds to the commit history
- **Cannot commit** to already accepted or rejected submissions

**Request fields:**
- `message` (optional) — Commit message describing what changed
- `title` (optional) — Submission title (only used on first commit)
- `description` (optional) — Detailed description of the submission

## SDLC Workflow

The Agent API follows an SDLC-style workflow where agent work goes through human review before reaching production:

```
┌─────────────────────────────────────────────────────────────┐
│                      FLOWS UI                                │
├─────────────────┬───────────────────┬───────────────────────┤
│   Production    │  Work in Progress │     Submissions       │
│                 │      (WIP)        │       (PRs)           │
├─────────────────┼───────────────────┼───────────────────────┤
│ Promoted        │ Active agent      │ Committed versions    │
│ versions        │ sessions          │ awaiting human review │
│ (is_production) │ (no submission)   │ (has submission)      │
└─────────────────┴───────────────────┴───────────────────────┘

Agent workflow:    active ──commit──> submitted ──[human accept]──> production
                                          │
                                    [human reject]
                                          ↓
                                      rejected (hidden)
```

### Submission Lifecycle

1. **Agent commits work** → Submission created with `status: pending`
2. **Human reviews** via web UI at `/submissions`
3. **Accept** → Version promoted to production, submission marked `accepted`
4. **Reject** → Submission marked `rejected`, hidden from agent view

## Plugin Development Workflow

When creating custom plugins for a flow, you MUST use the dedicated agent branch.

### CRITICAL: Always Use the Agent Branch

**NEVER commit to master or dev** unless explicitly requested. The agent branch from `/start` response is your isolated workspace.

```bash
# 1. Clone the repo
git clone <PLUGIN_REPO_URL>
cd <repo-directory>

# 2. CHECKOUT THE AGENT BRANCH (MANDATORY)
git checkout -b agent/abc-123   # Use branch.name from /start response

# 3. Create/modify plugins
mkdir my-plugin && cd my-plugin
# ... create plugin files ...

# 4. Commit and push TO THE AGENT BRANCH
git add my-plugin/
git commit -m "Add my-plugin"
git push origin agent/abc-123   # Push to agent branch, NOT master

# 5. Check build status (URL-encode the branch: agent/abc-123 -> agent%2Fabc-123)
curl -s -H "Authorization: Bearer $TOKEN" \
  "$BASE_URL/plugins/builds/<repo>/agent%2Fabc-123"

# 6. Wait for "status": "complete", then test
```

## Testing Plugins Directly

Test a plugin method without building a flow:

```bash
# IMPORTANT: URL-encode slashes in branch names
# agent/abc-123 -> agent%2Fabc-123
ENCODED_BRANCH=$(echo "$BRANCH" | sed 's/\//%2F/g')

curl -X POST "$BASE_URL/api/v1/agent/plugins/$ENCODED_BRANCH/sms/sendSms/execute" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "node_settings": {
      "provider": "twilio"
    },
    "global_settings": {
      "apiKey": "test-api-key"
    },
    "input": {
      "to": "+27821234567",
      "message": "Test message"
    }
  }'
```

URL format: `/api/v1/agent/plugins/:branch/:plugin/:method/execute`

> **Note:** Branch names containing `/` must be URL-encoded (replace `/` with `%2F`).
> Example: `agent/abc-123` → `agent%2Fabc-123`

**Request body fields:**
- `node_settings` (or `nodeSettings`) — Per-node config, available as `context.config` in plugin
- `global_settings` (or `globalSettings`) — Plugin-wide config, available as `context.pluginSettings` in plugin. Merged with any system-configured values.
- `input` — Method input data

Both snake_case and camelCase are accepted for all fields.

## Quick Reference

| Action | Method | Endpoint |
|--------|--------|----------|
| **Session Management** | | |
| Create new flow | POST | `/start/new` |
| Edit existing flow | POST | `/start/edit` (body: `flow_id`, `source_version_id`, `agent_id`) |
| **Flow Discovery** | | |
| Search/list flows | GET | `/flows?search=&tags=&limit=&offset=` |
| Get flow with versions | GET | `/flows/:flowId` |
| **Node Types** | | |
| List node types | GET | `/node-types/:branch` |
| **Flow Building** | | |
| Add node | POST | `/flows/:flowId/versions/:versionId/nodes` |
| Get node | GET | `/flows/:flowId/versions/:versionId/nodes/:nodeId` |
| Update node (partial) | PATCH | `/flows/:flowId/versions/:versionId/nodes/:nodeId` |
| Update node (replace) | PUT | `/flows/:flowId/versions/:versionId/nodes/:nodeId` |
| Delete node | DELETE | `/flows/:flowId/versions/:versionId/nodes/:nodeId` |
| Add edge | POST | `/flows/:flowId/versions/:versionId/edges` |
| Update edge (partial) | PATCH | `/flows/:flowId/versions/:versionId/edges/:edgeId` |
| Update edge (replace) | PUT | `/flows/:flowId/versions/:versionId/edges/:edgeId` |
| Delete edge | DELETE | `/flows/:flowId/versions/:versionId/edges/:edgeId` |
| Validate | GET | `/flows/:flowId/versions/:versionId/validate` |
| Get structure | GET | `/flows/:flowId/versions/:versionId/structure` |
| **Execution** | | |
| Insert item | POST | `/flows/:flowId/versions/:versionId/insert` (body: `node_id`, `payload`) |
| Run flow | POST | `/flows/:flowId/versions/:versionId/run` |
| Tick once | POST | `/flows/:flowId/versions/:versionId/tick` (body: optional `node_id`) |
| Stop flow | POST | `/flows/:flowId/versions/:versionId/stop` |
| **Monitoring** | | |
| Get status | GET | `/flows/:flowId/versions/:versionId/status` |
| Get logs | GET | `/flows/:flowId/versions/:versionId/logs` |
| Node items | GET | `/flows/:flowId/versions/:versionId/nodes/:nodeId/items` |
| Get work item | GET | `/work-items/:workId` |
| Get trace | GET | `/work-items/:workId/trace` |
| **Plugin Testing** | | |
| Execute plugin | POST | `/plugins/:branch/:plugin/:method/execute` |
| **Version Management** | | |
| Extend TTL | POST | `/versions/:versionId/extend` |
| **Commit & Submit** | | |
| Commit work | POST | `/versions/:versionId/commit` (body: `message`, `title`, `description`) |
