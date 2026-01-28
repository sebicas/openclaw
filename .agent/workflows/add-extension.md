---
description: Add a non-channel extension (tools, services, auth providers, memory backends) to Moltbot
---

# Add Extension Workflow

Use this workflow when adding a non-channel extension such as:
- **Tools** - Agent tools (e.g. `llm-task`, `lobster`)
- **Auth providers** - Model provider auth (e.g. `google-antigravity-auth`, `google-gemini-cli-auth`)
- **Memory backends** - Memory storage (e.g. `memory-core`, `memory-lancedb`)
- **Services** - Monitoring, proxy, diagnostics (e.g. `diagnostics-otel`, `copilot-proxy`)
- **Voice/media** - Non-channel integrations (e.g. `voice-call`)

For messaging channel extensions, use `add-channel-extension.md` instead.

---

## Step 1: Scaffold the extension directory

Create `extensions/<id>/` with these files:

### package.json

```json
{
  "name": "@moltbot/<id>",
  "version": "<match root package.json version>",
  "type": "module",
  "description": "Moltbot <name> plugin",
  "moltbot": {
    "extensions": ["./index.ts"]
  },
  "dependencies": {
    // Extension-specific runtime deps only
  },
  "devDependencies": {
    "moltbot": "workspace:*"
  }
}
```

**Key rules:**
- Version must match root `package.json` version exactly.
- Put `moltbot` in `devDependencies` (never `dependencies` with `workspace:*`).
- Runtime deps that are extension-specific go in `dependencies`.
- Do not add extension deps to the root `package.json` unless core uses them.
- Non-channel extensions omit the `channel` and `install` fields from `moltbot`.

### clawdbot.plugin.json

```json
{
  "id": "<id>",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  }
}
```

### CHANGELOG.md

```markdown
# @moltbot/<id>

## <version>

- Initial release
```

### index.ts

The entrypoint registers capabilities with the plugin API. Choose the registration
methods that match your extension type:

```typescript
import type { MoltbotPluginApi } from "clawdbot/plugin-sdk";
import { emptyPluginConfigSchema } from "clawdbot/plugin-sdk";

const plugin = {
  id: "<id>",
  name: "<Display Name>",
  description: "<Short description>",
  configSchema: emptyPluginConfigSchema(), // or a real schema
  register(api: MoltbotPluginApi) {
    // Pick the registration methods you need:

    // Tools - register agent tools
    // api.registerTool({ name: "...", description: "...", handler: ... });

    // Services - register background services
    // api.registerService({ name: "...", start: ..., stop: ... });

    // Hooks - register lifecycle hooks
    // api.registerHook("beforeReply", async (ctx) => { ... });

    // HTTP handlers - register webhook/API endpoints
    // api.registerHttpHandler(handleMyWebhook);

    // Gateway methods - register gateway RPC methods
    // api.registerGatewayMethod("myMethod", handler);

    // Config schema - if the extension has its own config section
    // (use a real schema instead of emptyPluginConfigSchema above)
  },
};

export default plugin;
```

---

## Step 2: Implement extension logic

### Extension type: Tool

For extensions that provide agent tools:

```typescript
import type { MoltbotPluginApi } from "clawdbot/plugin-sdk";

function register(api: MoltbotPluginApi) {
  api.registerTool({
    name: "tool_name",
    description: "What this tool does",
    inputSchema: {
      type: "object",
      properties: {
        param: { type: "string", description: "Parameter description" },
      },
      required: ["param"],
    },
    handler: async ({ input, context }) => {
      // Tool implementation
      return { result: "..." };
    },
  });
}
```

**Tool schema guardrails:**
- Avoid `Type.Union` in input schemas; no `anyOf`/`oneOf`/`allOf`
- Use `stringEnum`/`optionalStringEnum` for string enums
- Use `Type.Optional(...)` instead of `... | null`
- Top-level schema must be `type: "object"` with `properties`
- Avoid raw `format` property names (reserved keyword in some validators)

### Extension type: Auth Provider

For extensions that provide model provider authentication:

```typescript
function register(api: MoltbotPluginApi) {
  api.registerService({
    name: "<provider>-auth",
    start: async () => {
      // Initialize auth flow, token refresh, etc.
    },
    stop: async () => {
      // Cleanup
    },
  });
}
```

### Extension type: Memory Backend

For extensions that provide memory storage:

```typescript
function register(api: MoltbotPluginApi) {
  api.registerService({
    name: "<id>-memory",
    start: async () => {
      // Initialize storage connection
    },
    stop: async () => {
      // Close connections
    },
  });
}
```

### Extension type: Service/Integration

For extensions that provide background services, monitoring, or proxy functionality:

```typescript
function register(api: MoltbotPluginApi) {
  // Register HTTP endpoints if needed
  api.registerHttpHandler(async (req) => {
    if (req.path !== "/my-endpoint") return null;
    return { status: 200, body: { ok: true } };
  });

  // Register gateway methods if needed
  api.registerGatewayMethod("myExtension.status", async () => {
    return { status: "ok" };
  });
}
```

---

## Step 3: Add config schema (if needed)

If the extension needs user-configurable settings, create a real config schema
instead of using `emptyPluginConfigSchema()`:

```typescript
import { Type } from "@sinclair/typebox";

export const MyExtensionConfigSchema = Type.Object({
  enabled: Type.Optional(Type.Boolean()),
  apiKey: Type.Optional(Type.String()),
  endpoint: Type.Optional(Type.String()),
});
```

Update the `configSchema` field in the plugin object and in `clawdbot.plugin.json`.

---

## Step 4: Update GitHub labeler

Add a label entry to `.github/labeler.yml`:

```yaml
"extensions: <id>":
  - changed-files:
      - any-glob-to-any-file:
          - "extensions/<id>/**"
```

---

## Step 5: Add tests

Create colocated test files in `extensions/<id>/src/` or `extensions/<id>/`:
- Test tool handlers with mock contexts
- Test service start/stop lifecycle
- Test config schema validation
- Test HTTP handlers with mock requests

Use Vitest. Target 70% coverage.

---

## Step 6: Add documentation (if user-facing)

If the extension is user-installable or configurable, add docs:
- For plugins with CLI commands: `docs/plugins/<id>.md`
- For services: document in relevant section (e.g. `docs/providers/`)
- Update `docs/docs.json` navigation if adding a new page

**Doc rules:**
- Root-relative links, no `.md` suffix
- No personal hostnames/paths; use placeholders
- No em dashes or apostrophes in headings

---

## Step 7: Validate

```bash
# Lint
pnpm lint

# Type-check and build
pnpm build

# Run tests
pnpm test

# Verify plugin loads
pnpm moltbot plugins list
```

---

## Step 8: Commit

Use scoped commits:
```bash
scripts/committer "feat(<id>): add <name> extension" extensions/<id>/
scripts/committer "ci: add <id> extension label" .github/labeler.yml
```

Add a CHANGELOG entry.

---

## Reference: existing non-channel extensions

| Extension | Type | Key patterns |
|-----------|------|--------------|
| `voice-call` | Voice/telephony | Full config schema, HTTP webhooks, multiple providers (Twilio/Telnyx/Plivo), tunnel support |
| `memory-core` | Memory backend | Service registration, storage interface |
| `memory-lancedb` | Memory backend | External DB dependency, vector storage |
| `llm-task` | Tool | Agent tool registration, task orchestration |
| `lobster` | Feature | Integration with core UI/onboarding |
| `copilot-proxy` | Proxy | HTTP handler, request forwarding |
| `diagnostics-otel` | Monitoring | OpenTelemetry integration, service lifecycle |
| `google-antigravity-auth` | Auth provider | OAuth flow, token management |
| `google-gemini-cli-auth` | Auth provider | CLI-based auth, credential storage |
| `qwen-portal-auth` | Auth provider | Portal-based auth |
| `open-prose` | Feature | Text processing integration |

## Plugin API methods reference

The `MoltbotPluginApi` object passed to `register()` provides:

| Method | Use case |
|--------|----------|
| `api.runtime` | Access core runtime services |
| `api.pluginConfig` | Read plugin-specific config |
| `api.logger` | Plugin-scoped logger |
| `api.config` | Full app config |
| `api.registerChannel(...)` | Register a channel plugin (channels only) |
| `api.registerHttpHandler(...)` | Register HTTP/webhook endpoints |
| `api.registerGatewayMethod(...)` | Register gateway RPC methods |
| `api.registerTool(...)` | Register agent tools |
| `api.registerHook(...)` | Register lifecycle hooks |
| `api.registerService(...)` | Register background services |
