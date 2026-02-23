# CVM2 Workflow Automation Skill for Claude Code

Build and deploy automation flows in CVM2 through natural conversation. This Claude skill enables AI-powered workflow development — from simple notification flows to complex integrations with custom plugins.

## What You Can Do

**Build automation flows in seconds** — describe what you want automated, and Claude creates the flow structure, configures nodes, and tests execution.

```
"Create a flow that queries our database every hour and posts results to Slack"
"Build a workflow that sends an SMS when a new order comes in"
"Set up an automation that processes incoming webhooks and routes them based on content"
```

**Create custom plugins when you need new functionality** — Claude writes the plugin code, commits it to your repository, waits for the build, and tests the integration.

```
"Write a Slack plugin that posts formatted messages to our channels"
"Add a plugin that integrates with our CRM API"
"Build a plugin that calls our internal pricing service"
```

**Test and iterate rapidly** — insert test data, run flows step-by-step, inspect results, and debug issues without leaving the conversation.

## Installation

### 1. Download the skill

**Option A: Clone the repository**
```bash
git clone https://github.com/digitata/cvm
```

**Option B: Download ZIP from [Releases](https://github.com/digitata/cvm/releases)**

### 2. Install in Claude Code

1. Open Claude Code **Settings** (gear icon or `Cmd/Ctrl + ,`)
2. Navigate to **Capabilities** section
3. Scroll down to **Skills**
4. Click **Upload Skill** and select the `skills/cvm2-flow-automation` folder (or zip file)

### 3. Allow network access

In the same **Capabilities** section:
1. Find **Allow network egress** → **Domain allowlist**
2. Add your CVM2 base URL (e.g., `vaitom.digitata.ai`)

### 4. Gather your credentials

You'll need these from your CVM2 **Profile page** (top-right menu in the CVM2 GUI):

| Credential | Where to Find | When Needed |
|------------|---------------|-------------|
| **API Key** | Profile > API Keys | Always (format: `ak_...`) |
| **Base URL** | Your CVM2 domain | Always |
| **Git Clone URL** | Profile > API Keys | Only for custom plugins |

Claude will ask for these when you start working.

## Example Usage

### Real-World Example: Weekly Revenue Report to Slack

```
You: Create a CVM2 automation flow that connects to our Postgres database,
pulls revenue data from sales_revenue (sum price_usd by week using the
created_at column), compares the last 4 weeks with the previous 4 weeks,
and posts a summary to Slack. Use the dev_db pool and write a custom
Slack plugin.

Claude: I'll build this complete workflow for you...

[Creates development session with plugin branch]
[Writes custom Slack plugin with postMessage method]
[Commits, pushes, waits for build to complete]
[Tests the Slack plugin directly]

[Builds the flow:]
  - Ingress node (entry point)
  - PostgreSQL node querying weekly revenue aggregates
  - Custom Slack node posting the formatted comparison

[Inserts test data and runs the flow]
[Shows the Slack message preview with week-over-week analysis]
```

### Creating a Custom Plugin

```
You: I need a plugin that calls our inventory API to check stock levels

Claude: I'll create a custom plugin for this. Let me set up a development branch...
[Clones plugin repository, creates new plugin with proper structure]
[Writes the integration code with your API specifications]
[Commits, pushes, waits for build to complete]
[Tests the plugin method directly, then shows how to use it in flows]
```

### Building a Simple Flow

```
You: Create a flow that sends SMS notifications when triggered

Claude: I'll create this flow for you. First, I need your CVM2 credentials...
[Creates session, discovers available plugins, builds flow structure]
[Adds ingress node, connects SMS node with template configuration]
[Inserts test data and runs the flow]
[Shows results and confirms SMS was sent]
```

### Debugging a Flow

```
You: The flow isn't sending messages - can you check what's happening?

Claude: Let me inspect the flow execution...
[Checks flow logs and work item traces]
[Identifies configuration issue in the SMS node]
[Updates node config and re-runs test]
[Confirms successful execution]
```

## How It Works

This skill combines two CVM2 capabilities:

**Agent API** — Programmatic control over flow development:
- Create isolated development sessions
- Build flows by adding and connecting nodes
- Execute flows and monitor results
- Test individual plugin methods

**Plugin Service** — Custom code deployment:
- Write plugins in JavaScript/TypeScript
- Automatic build and deployment via Git
- Full access to Node.js ecosystem
- Vue components for custom node UIs

The skill includes comprehensive documentation that teaches Claude the CVM2 system, including:
- Complete API reference
- Plugin development patterns (methods, batch ingress, realtime ingress)
- Best practices and troubleshooting

## Available Node Types

CVM2 comes with plugins for common integrations:
- **SMS** — Send text messages via multiple providers
- **HTTP** — Make REST API calls
- **Database** — Query PostgreSQL, execute SQL
- **OpenAI** — LLM completions and embeddings
- And more via the plugin ecosystem

Discover available plugins by asking: *"What node types are available in my CVM2 instance?"*

## Requirements

- Claude Code CLI
- CVM2 instance with Agent API enabled
- API key with appropriate permissions
- (For custom plugins) Access to the plugin Git repository

## Troubleshooting

**"Plugin not showing after push"** — Wait for the build to complete. Claude checks build status automatically.

**"Build failed"** — Claude will fetch and show you the build logs with specific errors.

**"Flow not running"** — Claude inspects flow logs and work item traces to identify the issue.

**"API returns 401"** — Verify your API key is correct and hasn't expired. Check Profile > API Keys in CVM2.

## License

MIT

---

Built for [CVM2](https://vaitom.digitata.ai) — Flow-based automation platform
