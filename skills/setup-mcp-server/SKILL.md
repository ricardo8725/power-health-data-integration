---
name: setup-mcp-server
description: >
  Set up the health-import MCP server from scratch: scaffold the TypeScript project,
  install the MCP SDK, implement the three required tools (list, get, import), and
  connect it to Kiro. Use when initializing the MCP integration for a health tracking app.
---

# Set Up the health-import MCP Server

This skill walks you through creating a local MCP server that simulates an external
health data source (Apple Health, Garmin Connect) and exposes tools for importing
weight records into a tracking app.

---

## Prerequisites

Verify these are available before starting:

- **Node.js ≥ 18**: `node --version`
- **npm ≥ 9**: `npm --version`
- **TypeScript**: installed as a dev dependency (not globally required)

---

## Step 1: Scaffold the project

```bash
mkdir -p mcp-servers/health-import/src
mkdir -p mcp-servers/health-import/data
mkdir -p mcp-servers/health-import/dist
```

Create `mcp-servers/health-import/package.json`:

```json
{
  "name": "health-import-mcp",
  "version": "1.0.0",
  "type": "module",
  "main": "dist/index.js",
  "scripts": {
    "build": "tsc",
    "start": "node dist/index.js"
  },
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.0.0"
  },
  "devDependencies": {
    "typescript": "^5.4.0",
    "@types/node": "^20.0.0"
  }
}
```

Create `mcp-servers/health-import/tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "resolveJsonModule": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

```bash
cd mcp-servers/health-import && npm install
```

---

## Step 2: Create the synthetic dataset

Create `mcp-servers/health-import/data/sample-health-data.json`.

See `references/sample-data-schema.md` for the expected structure and a
template with ~15 synthetic records.

> ⚠️ Use only fictitious values. Add a `_meta.note` field that clearly labels
> the data as synthetic. Never commit real health measurements.

---

## Step 3: Implement the MCP server

Create `mcp-servers/health-import/src/index.ts` with these three tools:

### `list_sample_records`
Lists all available records with pre-calculated BMI. No input required.

### `get_record_by_id`
Returns full detail of a single record. Input: `{ id: string }`.

### `import_records_to_weight_tracking`
Returns records mapped to the WeightEntry schema, filtered by date range or
explicit id list. Input: `{ fromDate?: string, toDate?: string, ids?: string[] }`.
Output: array of `WeightEntryPayload` objects with `id: null` and `userId: null`.

See `references/mcp-server-template.md` for the complete implementation.

### Key implementation rules:
- Use `McpServer` (high-level API) from `@modelcontextprotocol/sdk/server/mcp.js`
- Use `StdioServerTransport` from `@modelcontextprotocol/sdk/server/stdio.js`
- Use `z.object({})` (not raw JSON Schema) for tools that take no arguments
- Log only to `process.stderr` — never to stdout (it pollutes the JSON-RPC channel)
- Keep BMI calculation local to the server (do not import from the main app)

---

## Step 4: Build and verify

```bash
cd mcp-servers/health-import
npm run build
```

Smoke test the server responds to the MCP initialize handshake:

```bash
echo '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"test","version":"0.0.1"}}}' \
  | node dist/index.js
```

Expected output contains `"serverInfo"` with your server name and version.

---

## Step 5: Connect to Kiro

Add the server to `~/.kiro/settings/mcp.json` under `mcpServers`:

```json
{
  "mcpServers": {
    "health-import": {
      "command": "node",
      "args": ["/absolute/path/to/mcp-servers/health-import/dist/index.js"],
      "env": {},
      "disabled": false
    }
  }
}
```

Reload Kiro's MCP connections from the MCP Server view in the Kiro panel.
Verify the three tools appear as available.

---

## Step 6: Add to .gitignore

```
mcp-servers/health-import/node_modules/
mcp-servers/health-import/dist/
```

The compiled `dist/` is excluded because it's reproducible with `npm run build`.
Commit `src/`, `data/`, `package.json`, `package-lock.json`, and `tsconfig.json`.
