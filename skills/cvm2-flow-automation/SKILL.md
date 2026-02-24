---
name: cvm2-development
description: End-to-end guide for developing custom functionality in CVM2
---

# CVM2 End-to-End Development

CVM2 is a flow-based automation platform. Flows are graphs of nodes that process work items. Each node executes a plugin method that transforms data or performs actions (send SMS, query database, call API, etc.).

This guide walks you through building automation flows and extending them with custom plugins.

## What You Can Do

1. **Build flows** using existing plugins (SMS, HTTP, Database, etc.)
2. **Create custom plugins** when you need new functionality
3. **Test everything** before deploying to production

## Before Starting

The following credentials are required. **If not provided, the agent should ask the user for them.** Users can find these in their **Profile page** (top right menu in the CVM2 GUI):

- **API Key** - For authentication (format: `ak_...`)
  - Found in Profile > API Keys section
  - Used with `Authorization: Bearer <api_key>` header

- **Base URL** - The CVM2 API endpoint
  - Usually the same domain as the GUI (e.g., `https://yourcvm.example.com`)

- **Plugin Repository Git URL** - Only needed when creating custom plugins
  - Found in Profile > API Keys section (Git Clone URL)
  - Used for `git clone` operations

---

## Building a Flow

Use this when you want to automate a process using existing plugins.

**More Information:** [Agent API Reference](agent-api.md)

### Step 1: Start a Session

**Create a NEW flow:**
```bash
curl -X POST "$BASE_URL/api/v1/agent/start/new" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "My Automation Flow",
    "agent_id": "session-001",
    "reason": "Building notification system",
    "ttl_minutes": 60
  }'
```

**Or EDIT an existing flow:**
```bash
# First, find the flow you want to edit
curl "$BASE_URL/api/v1/agent/flows?search=notification" \
  -H "Authorization: Bearer $TOKEN"

# Get details and see available versions
curl "$BASE_URL/api/v1/agent/flows/$FLOW_ID" \
  -H "Authorization: Bearer $TOKEN"

# Fork from a version to edit
curl -X POST "$BASE_URL/api/v1/agent/start/edit" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "flow_id": "existing-flow-uuid",
    "source_version_id": "production",
    "agent_id": "session-001",
    "reason": "Adding retry logic",
    "view_url": "https://example.com/workflow/abc-123/agent-session-001-..."
  }'
```

Save the `flow_id` and `version_id` from the response for subsequent API calls.
Both `/start/new` and `/start/edit` return a `view_url` field. This is a direct link to the flow editor where the user can watch the flow being built in real-time.  Show the `view_url` to the user immediately after starting a session.

### Step 2: See What Plugins Are Available

```bash
curl "$BASE_URL/api/v1/agent/node-types/$BRANCH_NAME" \
  -H "Authorization: Bearer $TOKEN"
```

> **Note:** `BRANCH_NAME` defaults to `dev`. Use your session's `branch.name` for custom plugins (URL-encode slashes: `agent/abc-123` → `agent%2Fabc-123`).

This returns all available node types. Look for the `id` field (e.g., `sms.sendSms`, `http.request`, `pg.sqlExecute`).

### Step 3: Add Nodes to Your Flow

Every flow needs an ingress node (entry point), then add the rest of your nodes:

```bash
curl -X POST "$BASE_URL/api/v1/agent/flows/$FLOW_ID/versions/$VERSION_ID/nodes" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"node_type": "ingress", "label": "Start"}'
```

Add the nodes you need:

```bash
curl -X POST "$BASE_URL/api/v1/agent/flows/$FLOW_ID/versions/$VERSION_ID/nodes" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "node_type": "sms.sendSms",
    "label": "Send Notification",
    "config": {
      "to": "{{phone}}",
      "message": "Hello {{name}}, your order is ready!"
    }
  }'
```

Save the `node_id` from each response.

### Step 4: Connect the Nodes

```bash
curl -X POST "$BASE_URL/api/v1/agent/flows/$FLOW_ID/versions/$VERSION_ID/edges" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "from_node": "<ingress-node-id>",
    "to_node": "<sms-node-id>"
  }'
```

### Step 5: Test It

Insert test data:

```bash
curl -X POST "$BASE_URL/api/v1/agent/flows/$FLOW_ID/versions/$VERSION_ID/insert" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "node_id": "<ingress-node-id>",
    "payload": {
      "phone": "+27821234567",
      "name": "John"
    }
  }'
```

Run the flow:

```bash
curl -X POST "$BASE_URL/api/v1/agent/flows/$FLOW_ID/versions/$VERSION_ID/run" \
  -H "Authorization: Bearer $TOKEN"
```

### Step 6: Check Results

```bash
curl "$BASE_URL/api/v1/agent/flows/$FLOW_ID/versions/$VERSION_ID/logs" \
  -H "Authorization: Bearer $TOKEN"
```

### Step 7: Submit for Review

When your flow is ready, commit your work to submit it for human review:

```bash
curl -X POST "$BASE_URL/api/v1/agent/versions/$VERSION_ID/commit" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "message": "Implemented notification flow",
    "title": "Order Notification System",
    "description": "Sends SMS when order is ready"
  }'
```

This creates a "submission" that a human reviewer can accept (promoting to production) or reject. You can continue making changes and committing until the submission is reviewed.

---

## Creating a Custom Plugin

Use this when existing plugins don't do what you need.

**More Information:** [Plugin Service Reference](plugin-service.md)

### Step 1: Start a Session with a Plugin Branch

```bash
curl -X POST "$BASE_URL/api/v1/agent/start/new" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Custom Plugin Development",
    "agent_id": "plugin-dev-001",
    "reason": "Creating CRM integration",
    "ttl_minutes": 120,
    "create_branch": true,
    "view_url": "https://example.com/workflow/abc-123/agent-session-001-...",

  }'
```

Save `flow_id`, `version_id`, and `branch.name` (e.g., `agent/abc-123`) for subsequent steps.
The response includes a `view_url` - send this to the user so they can watch your progress in the flow editor.

### Step 2: Clone the Plugin Repository

```bash
git clone <PLUGIN_REPO_URL>
cd plugins
git fetch origin
git checkout -b agent/abc-123 origin/master
```

Use the `branch.name` from Step 1.

### Step 3: Write Your Plugin

Create `plugins/plugin-myintegration/index.js`:

```javascript
module.exports = {
  name: 'myintegration',
  version: '1.0.0',

  methods: {
    fetchData: {
      name: 'Fetch Data',
      description: 'Get data from external API',

      nodeSettings: {
        apiKey: { type: 'string', required: true },
        endpoint: { type: 'string', required: true }
      },

      inputs: {
        recordId: { type: 'string' }
      },

      async execute(context, inputs) {
        const response = await fetch(
          `${context.config.endpoint}/${inputs.recordId}`,
          { headers: { 'Authorization': context.config.apiKey } }
        );
        return { data: await response.json() };
      }
    }
  }
};
```

### Step 4: Push and Wait for Build

```bash
git add .
git commit -m "Add myintegration plugin"
git push origin agent/abc-123
```

Check build status (wait for status `complete`):

```bash
curl -H "Authorization: Bearer $TOKEN" -s "$BASE_URL/plugins/builds/$PLUGINS_REPO/$(git rev-parse HEAD)"
{
    ...
    "status": "pending" | "running" | "complete" | "failed"
    ...
}
```

If build fails, check logs:

```bash
curl -H "Authorization: Bearer $TOKEN" -H "Accept: text/plain" "$BASE_URL/plugins/builds/$PLUGINS_REPO/$(git rev-parse HEAD)/logs"
```

### Step 5: Test the Plugin

Test your method directly before using it in a flow:

```bash
curl -X POST "$BASE_URL/api/v1/agent/plugins/agent%2Fabc-123/myintegration/fetchData/execute" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "node_settings": {
      "apiKey": "test-key-123",
      "endpoint": "https://api.example.com/records"
    },
    "input": {
      "recordId": "rec-456"
    }
  }'
```

### Step 6: Use It in a Flow

Once your plugin works, add it to your flow:

```bash
curl -X POST "$BASE_URL/api/v1/agent/flows/$FLOW_ID/versions/$VERSION_ID/nodes" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "node_type": "myintegration.fetchData",
    "label": "Get Record",
    "config": {
      "apiKey": "prod-key",
      "endpoint": "https://api.example.com/records"
    }
  }'
```

Then connect it and run the flow as shown in "Building a Flow" above.

### Step 7: Submit for Review

When your plugin and flow are ready, commit your work:

```bash
curl -X POST "$BASE_URL/api/v1/agent/versions/$VERSION_ID/commit" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "message": "Added CRM integration plugin",
    "title": "CRM Data Fetch Plugin",
    "description": "Fetches customer data from external CRM API"
  }'
```

A human reviewer will then accept (promoting to production) or reject your submission.

---

## Troubleshooting

**Plugin not showing up after push?**
Wait for the build to complete. Check status with the builds endpoint.

**Build failed?**
Check the build logs for JavaScript errors.

**Flow not running?**
Check the logs endpoint. Look for error messages in the response.

**Plugin method returning errors?**
Test it directly with the execute endpoint to see detailed error output.

**Errors during flow execution**
Check the status of the item you inserted. Also check the logs for the flow.

---

## Detailed Reference Documentation

- [Agent API Reference](agent-api.md) - All flow building endpoints
- [Plugin Service Reference](plugin-service.md) - Plugin code specification
