# health-data-integration — Kiro Power

> **Bonus Lesson 2 — Kiro University Challenge**
> Package and publish a reusable Kiro Power that integrates health data from
> wearables (Apple Health, Garmin Connect) into tracking apps via MCP,
> with safe synthetic demo data and built-in range validation.

[![Kiro Power](https://img.shields.io/badge/Kiro-Power-7c3aed?logo=data:image/svg+xml;base64,PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHZpZXdCb3g9IjAgMCAyNCAyNCI+PHBhdGggZmlsbD0id2hpdGUiIGQ9Ik0xMiAyTDIgN2wxMCA1IDEwLTV6TTIgMTdsOCA0IDgtNE0yIDEybDggNCA4LTQiLz48L3N2Zz4=)](https://kiro.dev/powers)
[![Agent Plugins](https://img.shields.io/badge/Agent%20Plugins-1.0.0-blue)](https://agent-plugins.org)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

---

## Background — Kiro University Challenge

This Power was built across two lessons of the [Kiro University Challenge](https://kiro.dev/university):

| Lesson | What was built |
|---|---|
| **Lesson 6 — MCP** | Local MCP server (`health-import`) with 3 tools, connected to Kiro via `~/.kiro/settings/mcp.json`. Demonstrated the full flow: MCP tools → domain validation → WeightEntry payloads. |
| **Bonus 2 — Package a Power** | The MCP server, steering conventions, and import skills packaged into this reusable Power following the [Agent Plugins spec](https://agent-plugins.org). Submitted to the Kiro Power registry. |

The source project is [HealthTrack](https://github.com/ricardo8725/healthtrack) — a personal
health tracking app built entirely with Kiro.

---

## What this Power does

This Power gives Kiro the knowledge and tools to:

1. **Set up a local MCP server** that simulates an external health data source
   (Apple Health exports, Garmin Connect records, CSV health data)
2. **Query that server** using three purpose-built MCP tools
3. **Validate every incoming record** against safe physiological ranges before reporting it as ready
4. **Map external records** to a `WeightEntry` schema, ready for `INSERT` into SQLite or Postgres

No real personal health data is ever used — the included dataset is 100% synthetic,
generated only to demonstrate the integration pattern.

---

## Quick start (no credentials needed)

The health-import MCP server runs locally with Node.js. No API keys, no cloud accounts.

```bash
# 1. Clone the repo
git clone https://github.com/ricardo8725/power-health-data-integration
cd power-health-data-integration

# 2. Build the MCP server
cd mcp-server
npm install
npm run build

# 3. Smoke test — should print the server startup message and respond to initialize
echo '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"test","version":"0.0.1"}}}' \
  | node dist/index.js
# Expected: ✅ health-import MCP server running via stdio
#           {"result":{"protocolVersion":"2024-11-05","capabilities":...}}

# 4. Test list_sample_records
echo '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"list_sample_records","arguments":{}}}' \
  | node dist/index.js
# Expected: 15 synthetic weight records with pre-calculated BMI
```

---

## Installation in Kiro

### Option A — Import from GitHub (recommended)
1. Open the Powers panel in Kiro
2. Click **Add Custom Power** → **Import from GitHub**
3. Paste: `https://github.com/ricardo8725/power-health-data-integration`

### Option B — Manual mcp.json entry

After building the server (`npm run build` inside `mcp-server/`), add this to
`~/.kiro/settings/mcp.json`:

```json
{
  "mcpServers": {
    "health-import": {
      "command": "node",
      "args": ["/absolute/path/to/power-health-data-integration/mcp-server/dist/index.js"],
      "env": {},
      "disabled": false
    }
  }
}
```

Replace `/absolute/path/to/` with the actual path where you cloned the repo.
Then reload MCP connections from the Kiro MCP Server panel.

> No placeholder credentials are needed for this server — it reads from a local
> JSON file and communicates over stdio.

---

## What's included

```
health-data-integration/
├── plugin.json                          # Power manifest (Agent Plugins 1.0.0 spec)
├── mcp.json                             # MCP server registration template
├── README.md                            # This file
├── dev.kiro/
│   └── steering/
│       └── health-data-conventions.md   # Always-on context: schema, validation rules, mapping
├── mcp-server/                          # Ready-to-build MCP server (TypeScript/Node)
│   ├── src/index.ts                     # Server implementation — 3 MCP tools
│   ├── data/sample-health-data.json     # 15 synthetic weight records (jul–sep 2026)
│   ├── package.json
│   └── tsconfig.json
└── skills/
    ├── setup-mcp-server/
    │   ├── SKILL.md                     # Step-by-step scaffolding guide
    │   └── references/
    │       ├── sample-data-schema.md    # Synthetic dataset structure & guidelines
    │       └── mcp-server-template.md   # Complete TypeScript implementation template
    └── import-health-data/
        ├── SKILL.md                     # Full import flow: query → validate → map → report
        └── references/
            └── validation-rules.md      # All validation rules with rejection format
```

---

## MCP tools

| Tool | Input | What it returns |
|---|---|---|
| `list_sample_records` | _(none)_ | All 15 synthetic records with pre-calculated BMI and category |
| `get_record_by_id` | `{ id: string }` | Full detail of one record (e.g. `"ext-007"`) |
| `import_records_to_weight_tracking` | `{ fromDate?, toDate?, ids? }` | WeightEntry payloads filtered by date range or explicit ids |

All tool responses with synthetic data include a `⚠️ SYNTHETIC DATA` disclaimer.
`id` and `userId` in returned payloads are always `null` — the app repository fills them before INSERT.

---

## WeightEntry schema (mapping target)

| Column | Type | Constraint |
|---|---|---|
| `id` | TEXT (UUID v4) | Generated by repository on INSERT — `null` in payloads |
| `user_id` | TEXT | Set by repository on INSERT — `null` in payloads |
| `date` | TEXT | `YYYY-MM-DD` |
| `weight_kg` | REAL | `0 < x ≤ 700` |
| `bmi` | REAL | Calculated, 2 decimal places |
| `bmi_category` | TEXT | `underweight` \| `normal` \| `overweight` \| `obese` |

---

## Validation rules (summary)

The Power enforces these rules on every record before marking it as ready for INSERT.
See `skills/import-health-data/references/validation-rules.md` for the full spec.

- `weightKg > 0` and `weightKg ≤ 700`
- `50 ≤ heightCm ≤ 300` (profile — rejects entire batch if invalid)
- `date` format `YYYY-MM-DD`, not more than 1 day in the future
- BMI recalculated locally and compared to source value (`|diff| < 0.01`)
- Category must match recalculated BMI using WHO thresholds

**Rejections are always explicit** — the agent never inserts silently.

---

## Skills

### `setup-mcp-server`
> Use when: "I need to create a health data MCP server from scratch."

Walks through scaffolding, SDK installation, implementing the three tools,
building, smoke testing, and connecting to Kiro. References include a
complete TypeScript implementation template.

### `import-health-data`
> Use when: "I want to import health records from the MCP server into my app."

Walks through querying the server, validating each record against physiological
ranges, mapping to WeightEntry payloads, reporting rejections clearly, and the
Drizzle ORM INSERT pattern.

---

## Privacy & data safety

- **Zero real health data** anywhere in this Power.
- All 15 records are completely fictitious — invented weight values over a 3-month period.
- The `_meta.note` field in `sample-health-data.json` explicitly labels every dataset as synthetic.
- The MCP server only logs to `stderr` (startup confirmation). stdout carries only JSON-RPC.
- No credentials, no API keys, no cloud dependencies required to run.

---

## Keywords that activate this Power

`health`, `apple health`, `garmin`, `wearable`, `weight tracking`, `bmi`,
`health data import`, `weight entry`, `health integration`, `synthetic data`, `mcp`

---

## Built with

- [Kiro](https://kiro.dev) — AI-powered development environment
- [@modelcontextprotocol/sdk](https://github.com/modelcontextprotocol/typescript-sdk) v1.30.0 — MCP TypeScript SDK
- [Agent Plugins spec](https://agent-plugins.org) v1.0.0 — Power packaging format
- [Kiro University Challenge](https://kiro.dev/university) — Lesson 6 (MCP) + Bonus 2 (Package a Power)

## License

MIT
