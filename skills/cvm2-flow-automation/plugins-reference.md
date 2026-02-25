# Plugin System Overview

This file provides guidance to Agents when working with code in the plugin repository.

## Development Commands

## Architecture Overview

This is a **vaitomcvm plugin system** that creates bundled ES modules compatible with the node.js processing engine and TypeScript-based plugins.

## Creating a new plugin

TO get started with the required boilerplate for creating a new plugin: 

```bash
./cvm2.sh create plugin --name myintegration
Created files:
  my-plugin/package.json
  my-plugin/plugin-my-plugin.mjs
```


## Extending a plugin

Simply change the plugin-*.mjs file with your new method / changes.


## Language Support

### TypeScript and JavaScript
- **TypeScript is PREFERRED** for new plugins - provides better type safety and development experience
- JavaScript is fully supported for simpler plugins
- The build system automatically detects TypeScript files and compiles them
- **TypeScript support is already configured** - just create `.ts` files and they will be built automatically

### Plugin Structure
Each plugin follows a standardized interface with **multiple methods support**:

#### Required Exports
- **`name`**: Plugin name (string) - must match directory name
- **`namespace`**: Plugin namespace (string) - usually same as name
- **`version`**: Plugin version (string) - semantic versioning
- **`description`**: Plugin description (string)
- **`author`**: Plugin author (string) 
- **`globalSettings`**: Global configuration schema
- **`methods`**: Object containing all plugin methods with metadata
- **`tags`**: Array of strings for categorization

#### Lifecycle Functions
- **`init(ctx)`**: **REQUIRED** - Initialize plugin with context (API keys, configuration, logging)
  - **CRITICAL**: Always read settings from `ctx.pluginSettings`, NOT `ctx.globalSettings`
  - Example: `const { pluginSettings } = ctx;` then use `pluginSettings.apiKey`
- **`close(ctx)`**: **REQUIRED** - Cleanup function for proper resource management
- **`[methodName](ctx, node, input)`**: Individual method implementations
  - **CRITICAL**: Always read node configuration from `node.config`, NOT `node.settings`
  - Example: `const { templateId } = node.config;`

### Naming Conventions

#### Directory and File Structure
```
plugin-name/
├── package.json           # Name: "plugin-name"  --  NB: this file has to be present for a directory build as plugin
├── plugin-name.mjs        # Main entry point
├── utility-files.ts       # TypeScript utility files (optional)
└── node_modules/          # Plugin-specific dependencies
```

#### Naming Rules
- **Directory name**: `kebab-case` (e.g., `qcontact`, `text-utils`)
- **Package name**: Must be `plugin-{directory-name}` (e.g., `plugin-qcontact`)
- **Main file**: Must be `plugin-{directory-name}.mjs` (e.g., `plugin-qcontact.mjs`)
- **Method names**: `camelCase` (e.g., `sendWhatsApp`, `sendSMS`)
- **Export names**: `camelCase` for functions, `UPPER_CASE` for constants
- package.json: plugins HAVE to have a package.json to be built by the build system.

### Metadata Schema

#### Global Settings Schema
```javascript
export const globalSettings = {
  settings: {
    settingName: {
      description: "Human readable description",
      defaultValue: "optional default", // optional
      default: "alternative to defaultValue" // optional
    }
  }
};
```

#### Methods Schema - **CRITICAL FOR NODE CONNECTIVITY**
```javascript
export const methods = {
  methodName: {
    name: "Human Readable Method Name",
    description: "What this method does",
    inputs: {
      paramName: {
        type: "string|number|boolean|array|object",
        description: "What this input parameter does/expects"
      }
    },
    outputs: {
      fieldName: {
        type: "string|number|boolean|array|object", 
        description: "What this output field contains"
      }
    },
    nodeSettings: {
      configParam: {
        description: "UI configuration description",
        defaultValue: "default value for UI",
        options: ["array", "of", "choices"], // optional for static dropdowns
        dynamicOptions: true // optional - enables dynamic dropdown population
      }
    },
    metadata: { // optional - processing engine hints
      maxConcurrency: 5,    // maximum concurrent executions
      maxTps: 2,           // maximum transactions per second
      maxBatchSize: 100    // maximum batch size for processing
    }
  }
};
```

**The `inputs` and `outputs` metadata is ESSENTIAL** - it describes what data flows between nodes and enables proper node graph connectivity in the UI.

## Plugin Method Metadata

Methods can include optional `metadata` for processing engine hints and performance tuning:

```javascript
metadata: {
  maxConcurrency: 5,    // Maximum concurrent executions of this method
  maxTps: 2,           // Maximum transactions per second (rate limiting)
  maxBatchSize: 100    // Maximum batch size for bulk processing
}
```

**Parameters:**
- **`maxConcurrency`**: Limits how many instances of this method can run simultaneously
- **`maxTps`**: Rate limits method execution to prevent API throttling or resource exhaustion
- **`maxBatchSize`**: For methods that process arrays, limits batch size for memory/performance control

**Use cases:**
- API methods with rate limits (e.g., WhatsApp, SMS sending)
- Database operations that need connection limiting
- CPU-intensive processing that should be throttled
- Memory-constrained operations requiring batch size limits

## Plugin Work Item Metadata Control

Plugins can control work item metadata fields (like active_at scheduling) by returning `{__metadata: {active_at: "ISO-date"}, payload: {...}}`. This enables dynamic scheduling, rate limiting, and business logic timing while maintaining full backward compatibility with existing plugins that return payload data directly. Plugin-set active_at overrides wait node timing, and the __metadata field is extensible for future work item metadata fields.

**Example:**
```javascript
// Return work item with scheduling metadata
return {
  __metadata: {
    active_at: new Date(Date.now() + 60000).toISOString() // Schedule 1 minute in future
  },
  payload: {
    // Your actual data
    id: 123,
    message: "Scheduled work item"
  }
};

// Backward compatible - plugins can still return just the payload
return {
  id: 123,
  message: "Immediate work item"
};
```

## Dynamic Options for Form Fields

Plugins can provide dynamic dropdown options that change based on the current form state and external data sources. See [DYNAMIC_OPTIONS.md](DYNAMIC_OPTIONS.md) for complete documentation.

### Quick Overview

1. **Mark field as dynamic** in nodeSettings:
```javascript
nodeSettings: {
  tableName: {
    description: "Table to query",
    dynamicOptions: true  // Enables dynamic options
  }
}
```

2. **Implement dynamic options function** using **EXACT** convention `dynamic_options_<methodName>_<fieldName>`:
```javascript
// CRITICAL: Function MUST be async and follow exact naming convention
export async function dynamic_options_queryTable_tableName(ctx, formValues) {
  const { connectionPool, schemaName } = formValues;
  // Fetch options based on current form values
  return await fetchTablesFromDatabase(connectionPool, schemaName);
}
```

**Convention Requirements:**
- Function MUST be `async`
- Function MUST be `export`ed
- Name MUST be `dynamic_options_<methodName>_<fieldName>`
- MUST return `Promise<string[]>`

3. **Backend integration** via POST `/plugin_options/:plugin/:method` endpoint

**Use cases:**
- Database tables/columns based on selected connection
- API endpoints based on authentication
- Context-aware formatting options
- Dependent dropdowns (country → state → city)

## Plugin Extension Capabilities - CRITICAL DISTINCTIONS

**🔥 CRITICAL RULE: Ingress nodes create new items "out of thin air" - anything processing existing items/work/batches should be a METHOD/ACTION!**

Plugins can expose three different types of extension capabilities:

### 1. Standard Processor (Method-based Processing) - "methods"

**Use for: Processing, transforming, filtering, batching, splitting, or manipulating existing data**

The most common type - processes input data and returns output data synchronously or asynchronously.

**✅ Examples of METHODS:**
- Transform text (uppercase, format, etc.)
- Filter arrays based on criteria
- Batch limit items (process N now, delay others)
- Split data into multiple outputs
- Validate/enrich existing data
- Call APIs with input data
- Database operations with input parameters
- Any processing of existing work items

**❌ NOT methods:**
- Polling external systems for new data
- Streaming from message queues
- Scheduled data loading

#### Metadata Schema
```javascript
export const methods = {
  methodName: {
    name: "Human Readable Method Name",
    description: "What this method does",
    inputs: {
      inputParam: {
        type: "string|number|boolean|array|object",
        description: "Input parameter description"
      }
    },
    outputs: {
      outputField: {
        type: "string|number|boolean|array|object",
        description: "Output field description"
      }
    },
    nodeSettings: {
      configParam: {
        description: "Configuration parameter",
        defaultValue: "default value"
      }
    }
  }
};
```

#### Implementation Pattern
```javascript
export async function methodName(ctx, node, input) {
  // Access configuration from node.config
  const configValue = node.config.configParam;

  // Access input data
  const inputValue = input?.inputParam;

  // Process and return output
  return {
    outputField: processedValue
  };
}
```

**Key characteristics:**
- Receives input data via the `input` parameter
- Returns output data as an object
- Executes when triggered by upstream nodes
- Synchronous or asynchronous processing

### 2. Batch Ingress (Scheduled Data Loading) - "batchIngress"

**Use for: Creating new work items from external sources on a schedule**

**🔥 KEY RULE: Does NOT receive input from other nodes - creates data "out of thin air"**

Loads data in batches from external sources on a schedule or manual trigger. Does NOT process incoming data from other nodes.

**✅ Examples of BATCH INGRESS:**
- Poll database for new records every N minutes
- Fetch data from REST APIs on schedule
- Load files from filesystem periodically
- ETL jobs that import external data
- Scheduled report generation
- Cron-like functionality

**❌ NOT batch ingress:**
- Processing existing work items
- Filtering/transforming data from upstream nodes
- Batch processing of received data (use METHOD instead)

#### Metadata Schema
```javascript
export const batchIngress = {
  methodName: {
    name: "Human Readable Batch Method Name",
    description: "What data this batch loader fetches",
    nodeSettings: {
      sqlQuery: {
        description: "SQL query to execute",
        required: true,
        defaultValue: "SELECT * FROM table"
      },
      maxRows: {
        description: "Maximum rows to fetch",
        defaultValue: 1000
      }
    },
    supportsScheduling: true  // Enables scheduling in UI
  }
};
```

#### Implementation Pattern
```javascript
export async function methodName(ctx, node, isManualExecution) {
  // Access configuration from node.config
  const query = node.config.sqlQuery;
  const maxRows = node.config.maxRows || 1000;

  // isManualExecution indicates if user triggered manually vs scheduled
  ctx.logger.info(`Batch ingress (manual: ${isManualExecution})`);

  // Fetch data from external source
  const rows = await fetchDataFromSource(query, maxRows);

  // Return array of work items
  return rows.map((row, index) => ({
    payload: {
      ...row,
      _metadata: {
        source: "batch-source",
        rowIndex: index,
        executedAt: new Date().toISOString()
      }
    }
  }));
}
```

**Key characteristics:**
- Does NOT receive `input` parameter (third param is `isManualExecution`)
- Returns array of work items: `[{ payload: {...} }, ...]`
- Executes on schedule or manual trigger
- Used for ETL, database queries, API polling, etc.
- Set `supportsScheduling: true` to enable scheduling in UI

### 3. Realtime Ingress (Streaming Data) - "realtimeIngress"

**Use for: Continuously streaming new work items from external sources**

**🔥 KEY RULE: Does NOT receive input from other nodes - creates streaming data "out of thin air"**

Streams data continuously from external sources in real-time using ReadableStream.

**✅ Examples of REALTIME INGRESS:**
- Stream from message queues (SQS, Kafka, RabbitMQ)
- WebSocket connections for live data
- File system watchers for new files
- Real-time event streams
- Continuous monitoring feeds
- Live API subscriptions

**❌ NOT realtime ingress:**
- Processing streams of existing data
- Transforming data from upstream nodes
- Stream processing of received items (use METHOD instead)

#### Metadata Schema
```javascript
export const realtimeIngress = {
  methodName: {
    name: "Human Readable Stream Name",
    description: "What this stream provides",
    nodeSettings: {
      queueUrl: {
        description: "Queue URL to poll",
        required: true
      },
      pollInterval: {
        description: "Polling interval in milliseconds",
        defaultValue: 1000
      },
      maxMessages: {
        description: "Max messages per poll",
        defaultValue: 10
      }
    }
  }
};
```

#### Implementation Pattern
```javascript
export async function methodName(ctx, node) {
  const queueUrl = node.config.queueUrl;
  const pollInterval = node.config.pollInterval || 1000;
  const maxMessages = node.config.maxMessages || 10;

  ctx.logger.info(`Starting realtime stream from ${queueUrl}`);

  return new ReadableStream({
    async start(streamController) {
      let isPolling = true;

      const poll = async () => {
        while (isPolling && !ctx.controller.signal.aborted) {
          try {
            // Poll external source
            const messages = await pollExternalSource(queueUrl, maxMessages);

            if (messages && messages.length > 0) {
              // Enqueue each message
              for (const message of messages) {
                streamController.enqueue({
                  payload: {
                    ...message,
                    source: queueUrl,
                    receivedAt: new Date().toISOString()
                  },
                  source: "realtime-source",
                  id: crypto.randomUUID()
                });
              }
            } else {
              // No messages, wait before next poll
              await new Promise(resolve => setTimeout(resolve, pollInterval));
            }
          } catch (error) {
            if (!ctx.controller.signal.aborted) {
              ctx.logger.error(`Polling error: ${error.message}`);
              await new Promise(resolve => setTimeout(resolve, pollInterval));
            }
          }
        }
      };

      // Start polling loop
      poll().catch(error => {
        ctx.logger.error(`Fatal polling error: ${error.message}`);
        streamController.error(error);
      });

      // Handle abort signal for cleanup
      ctx.controller.signal.addEventListener('abort', () => {
        ctx.logger.info('Stream abort signal received');
        isPolling = false;
        streamController.close();
      });
    },

    cancel() {
      ctx.logger.info("Stream cancelled");
    }
  });
}
```

**Key characteristics:**
- Returns a `ReadableStream` object
- Does NOT receive `input` parameter (only `ctx` and `node`)
- Continuously emits data via `streamController.enqueue()`
- Must respect `ctx.controller.signal.aborted` for cleanup
- Each enqueued item should have `{ payload, source, id }` structure
- Handle abort signal to clean up resources (timers, connections, etc.)

### 4. Dual-Purpose Node (Processor + Realtime Ingress)

A node can serve as BOTH a processor AND a realtime ingress source. This is useful when you want a node that can both stream data AND transform incoming data from other nodes.

#### Metadata Schema
```javascript
export const methods = {
  streamAndTransform: {
    name: "Stream and Transform",
    description: "Streams data in real-time AND transforms incoming data",

    // Realtime ingress configuration
    realtimeIngress: {
      name: "Stream Live Data",
      description: "Stream data in real-time",
      nodeSettings: {
        streamInterval: {
          description: "Stream interval in ms",
          defaultValue: 2000
        },
        streamSource: {
          description: "Source identifier",
          defaultValue: "live-feed"
        }
      }
    },

    // Normal processor inputs/outputs
    inputs: {
      value: {
        type: "any",
        description: "Value to transform"
      }
    },
    outputs: {
      transformed: {
        type: "any",
        description: "Transformed value"
      }
    },
    nodeSettings: {
      multiplier: {
        description: "Multiplier for numeric values",
        defaultValue: 2
      }
    }
  }
};
```

#### Implementation Pattern
```javascript
// Processor function - handles incoming data from other nodes
export async function streamAndTransform(ctx, node, input) {
  const multiplier = node.config.multiplier || 2;
  const value = input?.value;

  // Transform the input value
  let transformed;
  if (typeof value === "number") {
    transformed = value * multiplier;
  } else if (typeof value === "string") {
    transformed = value.repeat(multiplier);
  } else {
    transformed = value;
  }

  return { transformed };
}

// Realtime ingress function - streams data continuously
// Function name MUST be: {methodName}_realtimeIngress
export async function streamAndTransform_realtimeIngress(ctx, node) {
  const streamInterval = node.config.streamInterval || 2000;
  const streamSource = node.config.streamSource || "live-feed";

  ctx.logger.info(`Starting stream from ${streamSource}`);

  return new ReadableStream({
    async start(streamController) {
      let counter = 0;

      const intervalId = setInterval(() => {
        if (ctx.controller.signal.aborted) {
          clearInterval(intervalId);
          return;
        }

        streamController.enqueue({
          payload: {
            value: Math.floor(Math.random() * 100),
            streamSource,
            counter: ++counter,
            timestamp: new Date().toISOString()
          },
          source: "realtime-ingress",
          receivedAt: new Date().toISOString(),
          id: crypto.randomUUID()
        });
      }, streamInterval);

      ctx.controller.signal.addEventListener('abort', () => {
        clearInterval(intervalId);
        streamController.close();
      });
    },

    cancel() {
      ctx.logger.info("Stream cancelled");
    }
  });
}
```

**Key characteristics:**
- Two separate function implementations:
  - `methodName(ctx, node, input)` - processor function
  - `methodName_realtimeIngress(ctx, node)` - ingress function
- Metadata includes both `realtimeIngress` configuration AND `inputs/outputs`
- The node can be used either as a data source OR as a processor
- Both functions share the same `nodeSettings` configuration

### Quick Decision Guide

**🤔 Ask yourself: "Does this node receive data from upstream nodes?"**

- **YES** → It's a **METHOD** (even if it does batching, limiting, splitting, etc.)
- **NO** → It creates data from external sources → **Batch Ingress** or **Realtime Ingress**

**🤔 If creating data from external sources: "Is it continuous streaming?"**

- **YES** → **Realtime Ingress** (streams, queues, websockets, file watchers)
- **NO** → **Batch Ingress** (scheduled polls, ETL jobs, cron-like)

### Common Misconceptions

**❌ "Batch Limit" sounds like Batch Ingress**
- **✅ Correct**: It's a METHOD because it processes existing items from upstream nodes

**❌ "Stream Processing" sounds like Realtime Ingress**
- **✅ Correct**: It's a METHOD because it processes existing stream data from upstream nodes

**❌ "Data Loading" sounds like Ingress**
- **✅ Correct**: Only if loading from external sources. If processing data passed from upstream nodes, it's a METHOD

**Remember: Ingress = "out of thin air", Methods = "processing existing data"**

## Vue Component GUI System

Plugins can define **custom Vue components** for rich node UIs in the workflow canvas. Components are automatically compiled during the build process.
The plugin components are rendered in an environment where Vue 3 BootstrapVue (based on Bootstrap v4) is available.

### Styling Guidelines

Prefer Bootstrap v4 classes over custom styles.

- **Use Bootstrap v4 utility classes** provided by BootstrapVue for styling (e.g., `text-muted`, `small`, `d-flex`, `mb-2`)
- **Avoid custom `<style>` blocks** unless absolutely necessary for unique styling requirements

### Convention-Based Component Discovery

Create a `.vue` file in the plugin directory matching the method name:

```
digitata/
├── plugin-digitata.mjs          # Plugin code
├── delay.vue                    # Component for delay method
├── escalateToCallCentre.vue     # Component for escalateToCallCentre method
└── flowController.vue           # Component for flowController method
```

### Component Structure

Components receive two props and can emit events:

```vue
<template>
  <div class="my-node">
    <div class="node-header">
      <span class="node-title">{{ methodName }}</span>
    </div>
    <div class="setting-row">
      <label>Setting:</label>
      <span>{{ nodeData.config.mySetting }}</span>
    </div>
    <button @click="handleAction">Test</button>
  </div>
</template>

<script>
export default {
  props: {
    nodeData: {      // Node configuration and state
      type: Object,
      required: true
    },
    nodeId: {        // Unique node identifier
      type: String,
      required: true
    }
  },
  computed: {
    methodName() {
      return this.nodeData?.config?.methodName || 'Unknown';
    }
  },
  methods: {
    handleAction() {
      // Emit events to trigger backend actions
      this.$emit('plugin-action', {
        type: 'test-connection',
        nodeId: this.nodeId
      });
    }
  }
}
</script>
```

### Build Output

The build system compiles `.vue` files into self-contained IIFE bundles:

**Generated Files:**
- `dist/components/{pluginName}/plugin-{pluginName}-{methodName}.component.js`
- Component metadata included in `dist/plugins-metadata.json`
- All components packaged in `dist/plugin-components.zip`

**Component Format:**
- IIFE bundles that expect Vue as a global variable
- Support CommonJS, AMD, and browser globals
- Export to `window.{pluginName}_{methodName}`

**Example:**
```javascript
// dist/components/digitata/plugin-digitata-delay.component.js
(function(Vue) {
  'use strict';

  // Compiled render function
  function render(_ctx, _cache) { /* ... */ }

  // Component options
  const componentOptions = { /* props, methods, computed */ };

  // Export
  const component = { name: 'delay', render, ...componentOptions };
  window.digitata_delay = component;
  return component;
})(window.Vue);
```

### Frontend Integration

Components can be dynamically loaded in the Vue application:

```javascript
// Load component script
const script = document.createElement('script');
script.src = releaseUrl + '/components/digitata/plugin-digitata-delay.component.js';
document.head.appendChild(script);

// Use component (available as window.digitata_delay)
app.component('digitata-delay-node', window.digitata_delay);
```

### Component Metadata

Component information is automatically included in `plugins-metadata.json`:

```json
{
  "plugins": {
    "digitata": {
      "components": [
        {
          "name": "delay",
          "path": "components/digitata/plugin-digitata-delay.component.js"
        }
      ]
    }
  }
}
```

**📚 For complete integration guide, see [VUE_COMPONENTS_GUIDE.md](./VUE_COMPONENTS_GUIDE.md)**

### Build System
The build system uses **Rollup** with custom configuration to:
1. **Detect TypeScript** - Only includes TypeScript plugin if `.ts` files are present
2. **Compile Vue Components** - Converts `.vue` SFCs into IIFE bundles
3. Bundle Node.js dependencies for Deno compatibility
4. Transform CommonJS modules to ES modules
5. Rewrite bare Node built-ins to `node:*` format (e.g., `events` → `node:events`)
6. Handle external dependencies like `pg-native` that can't be bundled
7. Add global polyfills for `Buffer` and `process` in the bundle output

### Plugin Examples
- **pg**: PostgreSQL database plugin using `pg` client library
- **openai**: OpenAI API integration plugin 
- **qcontact**: QContact messaging plugin (TypeScript) - WhatsApp & SMS with multiple methods
- **text-utils**: Text processing utilities with multiple methods (analyze, transform, extract)
- **day-of-week**: Simple date processing plugin (no external dependencies)

### Key Files
- `rollup.config.mjs`: Main Rollup configuration with Deno compatibility transforms
- `Makefile`: Build automation for all plugins
- `host.ts`: Deno test runner for plugins
- `deno_load_plugin.ts`: Alternative Deno loader
- `build-plugin.sh`: Individual plugin build script
- `import-config.mjs`: Legacy configuration file

### External Dependencies
Plugins can use Node.js libraries, but they must be bundleable. The build system:
- Keeps Node built-ins external (accessed via `node:*`)
- Bundles npm dependencies into the output
- Handles CJS to ESM conversion automatically
- Provides global `Buffer` and `process` polyfills

### Development Workflow
1. Create plugin directory with `package.json` and `plugin-[name].mjs`
2. Install dependencies: `make install` 
3. **Build is handled by the user** - you don't need to run build commands after making changes
4. Test with Deno: `deno run --allow-net --allow-read host.ts` (if needed)

### Security Considerations
- API keys are passed through context in `init()` function
- External dependencies are bundled to avoid runtime resolution
- Native addons (`.node` files) are explicitly forbidden

### General Guidance

- **Always commit changes** after making modifications - use relevant commit messages
- **Do NOT run build commands** - the user will handle building and testing
- Focus on code changes and commit them immediately when complete

### CRITICAL Implementation Patterns

When implementing plugins, follow these exact patterns from the working qcontact plugin:

#### Settings Access in init() Function
```javascript
export async function init(ctx) {
  const { pluginSettings } = ctx;  // CORRECT - use pluginSettings
  
  if (!pluginSettings.apiKey) {     // CORRECT - read from pluginSettings
    throw new Error('API key is required');
  }
  
  // Initialize client with pluginSettings values
  client = new ApiClient(pluginSettings.apiKey, pluginSettings.baseUrl);
}
```

**DO NOT USE**: `ctx.globalSettings` - this is WRONG

#### Node Configuration Access in Method Functions  
```javascript
export async function methodName(ctx, node, input) {
  const templateId = node.config.templateId;        // CORRECT - use node.config
  const baseUrl = node.config.baseUrl;             // CORRECT - use node.config
  const customParam = node.config.customParam;     // CORRECT - use node.config
  
  // Use the configuration values...
}
```

**DO NOT USE**: `node.settings` - this is WRONG

These patterns are essential for proper plugin functionality. Any deviation will cause runtime errors.
