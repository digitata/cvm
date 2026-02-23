---
name: cvm2-custom-plugins
description: Write code to extend cvm2
version: 1.0.0
---

# CVM2 Custom Plugin Service

Write any custom code that extends the functionality available in cvm2. Plugins are javascript extensions that extend the available nodes
for use in flows.

## Overview

A custom Git Repo is available at <%= base_url %>. For detailed plugin structure, naming conventions, and implementation patterns, refer to the [Plugin Reference Guide](plugins-reference.md).

The Plugin Build Service is an HTTP API that compiles plugin source code into production-ready bundles. It supports:

**Base URL:**
- Local: `http://localhost:80` (via nginx) or `http://localhost:3000` (direct)
- ECS/ALB: `https://<alb-dns>/plugins` (all endpoints prefixed with `/plugins`)


## Source Repository

The Source Repo will be a git repository that will be provided by the user. Ask the user to provide the full git clone url if not available.

**Important:** The cloned repository contains a `CLAUDE.md` file with comprehensive documentation on plugin development, including:
- Plugin structure and naming conventions
- Method, Batch Ingress, and Realtime Ingress patterns
- Vue component GUI system
- Dynamic options for form fields
- Build system details

After cloning, read the `CLAUDE.md` file in the repository root for the most up-to-date plugin development guidance. A copy of this documentation is also available in the [Plugin Reference Guide](plugins-reference.md).

## Workflow

Copy this workflow to make a change or create a new plugin:

```
Cvm2 Customize Functionality Checklist
 - [ ] Clone the repository
 - [ ] Make code changes
 - [ ] Commit and push
 - [ ] Wait for build
 - [ ] Test and Verify

**Step 1: Clone the repository**

Clone the repository provided by the user:

```bash
git clone <USER_PROVIDED_CLONE_URL>
cd <repo-directory>
```

After cloning, read the `CLAUDE.md` file in the repository root for detailed plugin development guidance including structure, patterns, and examples.

**Step 2: Make code changes**

Modify or add plugin code according to the [Plugin Reference Guide](plugins-reference.md) (or `CLAUDE.md` in the cloned repo).

Typical actions include:

Editing existing plugin modules

Creating new plugin modules

Updating plugin metadata or configuration

**Step 3: Commit and push**

All modifications must be committed to the repository.
Use a clear commit message that describes the plugin change or feature.
Pushing automatically triggers the Plugin Build Service to compile the plugins.

**Step 4: Wait for Build**
The following API call shows the status of the build:
```bash
curl -H "Authorization: Bearer $TOKEN" -s http://$BASE_URL/plugins/builds/<my-plugins>/<commit>
```
It is suggested to sleep in increments of 15s while waiting for build. Status will be "complete" when it is done. "failed" when error.
Make sure to use the correct repo name and commit id, else you will get 404.

If an error occurs, the build logs endpoint can be used to diagnose build and code problems:

```bash
curl -H "Authorization: Bearer $TOKEN" -H "Accept: text/plain" http://$BASE_URL/plugins/builds/<repo>/<commit>/logs
```

**Step 5: Test and Verify**

As part of the Agent API, a "method execute" endpoint is provided to test an arbitrary plugin/method
with payloads:

```bash
curl -X POST ""http://$BASE_URL/api/v1/agent/plugins/agent%2Fabc-123/sms/sendSms/execute"
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    -d '{
      "node_settings": {
        "provider": "twilio"
      },
      "input": {
        "to": "+27821234567",
        "message": "Test message"
      }
    }'
```

The format of URL is: /api/v1/agent/plugins/:branch/:plugin/:method/execute
