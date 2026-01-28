---
description: Add a new messaging channel as a Moltbot extension plugin
---

# Add Channel Extension Workflow

Use this workflow when adding a new messaging platform (e.g. Viber, Rocket.Chat, IRC)
as a channel extension under `extensions/`.

## Prerequisites

- Read `CLAUDE.md` (especially the "Extension Development" section)
- Identify the channel's API type: Bot API (token-based), OAuth, QR link, WebSocket, webhook, etc.
- Determine the channel ID (lowercase, no spaces, e.g. `viber`, `rocketchat`)
- Check that `extensions/<id>/` does not already exist

---

## Step 1: Scaffold the extension directory

Create `extensions/<id>/` with these files:

### package.json

```json
{
  "name": "@moltbot/<id>",
  "version": "<match root package.json version>",
  "type": "module",
  "description": "Moltbot <Name> channel plugin",
  "moltbot": {
    "extensions": ["./index.ts"],
    "channel": {
      "id": "<id>",
      "label": "<Display Name>",
      "selectionLabel": "<Name> (<Protocol>)",
      "docsPath": "/channels/<id>",
      "docsLabel": "<id>",
      "blurb": "Short description for onboarding selection.",
      "aliases": ["<short-alias>"],
      "order": 80,
      "quickstartAllowFrom": true
    },
    "install": {
      "npmSpec": "@moltbot/<id>",
      "localPath": "extensions/<id>",
      "defaultChoice": "npm"
    }
  },
  "devDependencies": {
    "moltbot": "workspace:*"
  }
}
```

**Key rules:**
- Version must match root `package.json` version exactly.
- Put `moltbot` in `devDependencies` (never `dependencies` with `workspace:*`).
- Platform-specific SDKs go in `dependencies`.
- `order` controls onboarding list position (lower = higher).
- Set `quickstartAllowFrom: true` if the channel supports DM pairing.

### clawdbot.plugin.json

```json
{
  "id": "<id>",
  "channels": ["<id>"],
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

```typescript
import type { MoltbotPluginApi } from "clawdbot/plugin-sdk";
import { emptyPluginConfigSchema } from "clawdbot/plugin-sdk";

import { <id>Plugin } from "./src/channel.js";
import { set<Name>Runtime } from "./src/runtime.js";

const plugin = {
  id: "<id>",
  name: "<Display Name>",
  description: "<Display Name> channel plugin",
  configSchema: emptyPluginConfigSchema(),
  register(api: MoltbotPluginApi) {
    set<Name>Runtime(api.runtime);
    api.registerChannel({ plugin: <id>Plugin });
    // api.registerHttpHandler(handle<Name>Webhook); // if webhook-based
  },
};

export default plugin;
```

---

## Step 2: Implement the runtime singleton

Create `extensions/<id>/src/runtime.ts`:

```typescript
import type { PluginRuntime } from "clawdbot/plugin-sdk";

let runtime: PluginRuntime | null = null;

export function set<Name>Runtime(r: PluginRuntime): void {
  runtime = r;
}

export function get<Name>Runtime(): PluginRuntime {
  if (!runtime) throw new Error("<Name> plugin runtime not initialized");
  return runtime;
}
```

---

## Step 3: Implement the channel plugin

Create `extensions/<id>/src/channel.ts` implementing `ChannelPlugin`.

Reference existing extensions for the pattern:
- **Simple channel:** `extensions/zalo/src/channel.ts` (~400 LOC)
- **Moderate channel:** `extensions/line/src/channel.ts` (~500 LOC)
- **Complex channel:** `extensions/msteams/src/channel.ts`

### Required adapters

At minimum implement these adapters on the `ChannelPlugin` object:

1. **`meta`** - Channel metadata (id, label, docsPath, blurb, aliases)
2. **`capabilities`** - Feature flags:
   ```typescript
   capabilities: {
     chatTypes: ["direct"],        // or ["direct", "group"]
     media: true,                  // supports media attachments
     reactions: false,             // supports emoji reactions
     threads: false,               // supports threaded replies
     polls: false,                 // supports polls
     nativeCommands: false,        // supports slash commands
     blockStreaming: true,         // buffer full response before sending
   }
   ```
3. **`config`** - Account management:
   - `listAccountIds(cfg)` - List configured account IDs
   - `resolveAccount(cfg, accountId)` - Resolve account config + credentials
   - `defaultAccountId(cfg)` - Get the default account ID
   - `isConfigured(account)` - Check if credentials are present
   - `describeAccount(account)` - Return `ChannelAccountSnapshot`
   - `resolveAllowFrom(...)` - Get allowlist entries
   - `formatAllowFrom(...)` - Normalize allowlist entries
   - `setAccountEnabled(...)` - Enable/disable account in config
   - `deleteAccount(...)` - Remove account from config
4. **`security`** - DM access policy:
   - `resolveDmPolicy(...)` - Return policy, allowFrom, config paths
5. **`outbound`** - Message sending:
   - `sendText(...)` - Send a text message
   - `sendMedia(...)` - Send media with optional caption
   - `chunker(text, limit)` - Split long messages
   - `textChunkLimit` - Max characters per message
   - `deliveryMode` - Usually `"direct"`
6. **`status`** - Status and diagnostics:
   - `collectStatusIssues(...)` - Return config/credential issues
   - `probeAccount(...)` - Health check the bot/connection
   - `buildAccountSnapshot(...)` - Build status snapshot
   - `buildChannelSummary(...)` - Build channel-level summary
   - `defaultRuntime` - Initial runtime state
7. **`gateway`** - Gateway lifecycle:
   - `startAccount(ctx)` - Start monitoring (polling/webhook)

### Optional adapters (add as needed)

- **`onboarding`** - CLI setup wizard
- **`setup`** - Account setup (resolveAccountId, validateInput, applyAccountConfig)
- **`pairing`** - DM pairing flow (idLabel, normalizeAllowEntry, notifyApproval)
- **`groups`** - Group message handling (resolveRequireMention)
- **`messaging`** - Target normalization for `moltbot message send`
- **`directory`** - Peer/group listing
- **`threading`** - Thread/reply handling
- **`actions`** - Message action buttons
- **`agentPrompt`** - Agent prompt hints for rich messages
- **`mentions`** - @mention handling

### Supporting files

Create as needed:
- `src/config-schema.ts` - Zod schema for channel config section
- `src/accounts.ts` - Account resolution helpers (listIds, resolve, resolveDefault)
- `src/send.ts` - Message sending logic (API calls)
- `src/monitor.ts` - Inbound message monitor (polling loop or webhook handler)
- `src/probe.ts` - Bot info / health check endpoint
- `src/status-issues.ts` - Status diagnostic checks
- `src/actions.ts` - Message actions
- `src/onboarding.ts` - Onboarding adapter

---

## Step 4: Write documentation

Create `docs/channels/<id>.md` following this structure:

```markdown
---
summary: "<Name> setup, configuration, and usage"
read_when:
  - You want to connect Moltbot to <Name>
  - You need <Name> credential setup
---

# <Name> (plugin)

<Status and capability summary.>

## Plugin required

Install the <Name> plugin:

\`\`\`bash
moltbot plugins install @moltbot/<id>
\`\`\`

Local checkout:

\`\`\`bash
moltbot plugins install ./extensions/<id>
\`\`\`

## Setup

<Step-by-step setup instructions with the platform's developer console.>

## Configure

<Minimal config example in json5.>

## How it works

<Behavior: inbound handling, routing, DM policy, groups.>

## Troubleshooting

<Common issues and fixes.>
```

**Doc rules:**
- Root-relative links, no `.md` suffix: `[Plugins](/plugin)`
- No personal hostnames/paths; use `gateway-host` placeholder
- No em dashes or apostrophes in headings (breaks Mintlify anchors)

---

## Step 5: Update docs navigation

Add the new page to `docs/docs.json` in the Channels group:

```json
{
  "group": "Channels",
  "pages": [
    "channels/index",
    ...
    "channels/<id>",
    ...
  ]
}
```

Place it in a logical position (alphabetical among plugin channels, after core channels).

---

## Step 6: Update GitHub labeler

Add a label entry to `.github/labeler.yml`:

```yaml
"channel: <id>":
  - changed-files:
      - any-glob-to-any-file:
          - "extensions/<id>/**"
          - "docs/channels/<id>.md"
```

---

## Step 7: Add tests

Create colocated test files in `extensions/<id>/src/`:
- `channel.test.ts` - Plugin registration, adapter contracts
- `send.test.ts` - Message sending (mock HTTP)
- `monitor.test.ts` - Inbound message processing
- `probe.test.ts` - Health check

Use Vitest. Target 70% coverage.

---

## Step 8: Validate

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

## Step 9: Commit

Use scoped commits:
```bash
scripts/committer "feat(<id>): add <Name> channel extension" extensions/<id>/ docs/channels/<id>.md
scripts/committer "docs: add <Name> to channel navigation" docs/docs.json .github/labeler.yml
```

Add a CHANGELOG entry with the new channel.

---

## Reference: existing extensions by complexity

| Complexity | Extension | Key patterns |
|------------|-----------|--------------|
| Minimal    | `zalo`    | Bot API, polling + webhook, DMs only |
| Simple     | `line`    | Webhook-only, rich messages (Flex), groups |
| Moderate   | `matrix`  | E2EE, SDK-based, room management |
| Complex    | `msteams` | Bot Framework, Express server, enterprise auth |
| Complex    | `discord` | Gateway WebSocket, slash commands, threads |
