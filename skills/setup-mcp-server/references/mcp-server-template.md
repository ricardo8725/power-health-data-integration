# MCP Server Implementation Template

Complete `src/index.ts` for the health-import MCP server.

```typescript
import { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
import { readFileSync } from 'node:fs';
import { fileURLToPath } from 'node:url';
import { dirname, join } from 'node:path';
import { z } from 'zod';

// ── Dataset ──────────────────────────────────────────────────────────────────
const __filename = fileURLToPath(import.meta.url);
const DATA_PATH = join(dirname(__filename), '..', 'data', 'sample-health-data.json');

interface RawRecord {
  id: string; date: string; weightKg: number; source: string; notes: string | null;
}
interface DataFile {
  _meta: { note: string };
  profile: { heightCm: number; dateOfBirth: string };
  records: RawRecord[];
}

const { records, profile } = JSON.parse(readFileSync(DATA_PATH, 'utf-8')) as DataFile;

// ── BMI (local — do not import from main app) ─────────────────────────────
type BMICategory = 'underweight' | 'normal' | 'overweight' | 'obese';

function calculateBMI(w: number, h: number) {
  const v = Math.round(w / ((h / 100) ** 2) * 100) / 100;
  const c: BMICategory = v < 18.5 ? 'underweight' : v < 25 ? 'normal' : v < 30 ? 'overweight' : 'obese';
  return { value: v, category: c };
}

// ── WeightEntry payload ────────────────────────────────────────────────────
function toEntry(r: RawRecord) {
  const { value, category } = calculateBMI(r.weightKg, profile.heightCm);
  return { id: null, userId: null, date: r.date, weightKg: r.weightKg,
           bmi: value, bmiCategory: category, sourceSystem: r.source };
}

// ── Server ─────────────────────────────────────────────────────────────────
const server = new McpServer({ name: 'health-import', version: '1.0.0' });

server.registerTool('list_sample_records',
  { description: 'Lists all synthetic records with pre-calculated BMI.',
    inputSchema: z.object({}) },
  async () => ({
    content: [{ type: 'text' as const,
      text: JSON.stringify({ total: records.length, profileHeightCm: profile.heightCm,
        records: records.map(r => ({ ...r, ...calculateBMI(r.weightKg, profile.heightCm) })) }, null, 2) }]
  })
);

server.registerTool('get_record_by_id',
  { description: 'Returns detail of a single record by id.',
    inputSchema: z.object({ id: z.string() }) },
  async ({ id }) => {
    const r = records.find(x => x.id === id);
    if (!r) return { content: [{ type: 'text' as const, text: JSON.stringify({ error: `Not found: ${id}` }) }], isError: true };
    return { content: [{ type: 'text' as const,
      text: JSON.stringify({ ...r, ...calculateBMI(r.weightKg, profile.heightCm), profileHeightCm: profile.heightCm }, null, 2) }] };
  }
);

server.registerTool('import_records_to_weight_tracking',
  { description: 'Returns WeightEntry payloads filtered by date range or ids.',
    inputSchema: z.object({
      fromDate: z.string().regex(/^\d{4}-\d{2}-\d{2}$/).optional(),
      toDate:   z.string().regex(/^\d{4}-\d{2}-\d{2}$/).optional(),
      ids:      z.array(z.string()).optional(),
    }) },
  async ({ fromDate, toDate, ids }) => {
    const filtered = ids?.length
      ? records.filter(r => ids.includes(r.id))
      : records.filter(r => (!fromDate || r.date >= fromDate) && (!toDate || r.date <= toDate));
    if (!filtered.length) return {
      content: [{ type: 'text' as const, text: JSON.stringify({ error: 'No records matched.' }) }], isError: true };
    return { content: [{ type: 'text' as const,
      text: JSON.stringify({ count: filtered.length,
        note: '⚠️ SYNTHETIC DATA. id and userId are null — fill before INSERT.',
        weightEntries: filtered.map(toEntry) }, null, 2) }] };
  }
);

// ── Start ──────────────────────────────────────────────────────────────────
const transport = new StdioServerTransport();
await server.connect(transport);
process.stderr.write('✅ health-import MCP server running\n');
```
