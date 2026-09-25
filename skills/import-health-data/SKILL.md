---
name: import-health-data
description: >
  Import weight records from an external health data source (via the health-import
  MCP server) into a WeightEntry table. Covers querying the MCP tools, validating
  each record against safe physiological ranges, mapping to the WeightEntry schema,
  and reporting rejections explicitly. Use when importing wearable or health export
  data into a weight tracking app.
---

# Import Health Data via MCP

This skill guides you through the complete import flow: query the MCP server,
validate incoming records, map them to the WeightEntry schema, and surface any
rejections to the user before inserting.

---

## Prerequisites

- The `health-import` MCP server is running and connected in Kiro
- The target app has a `weight_entry` table matching the WeightEntry schema
- The user's profile (`height_cm`) is available

---

## Step 1: Query the MCP server

Use one of the three available tools depending on what the user needs:

```
# List everything available
→ list_sample_records   (no input)

# Get one specific record
→ get_record_by_id      ({ id: "ext-007" })

# Get records by date range or explicit ids
→ import_records_to_weight_tracking({
    fromDate: "2026-09-01",   // optional
    toDate:   "2026-09-22",   // optional
    ids:      ["ext-001"]     // optional — overrides date range
  })
```

---

## Step 2: Validate every record before reporting it as ready

Apply all rules from `references/validation-rules.md`. Never skip validation
even if the source is trusted.

**Always separate the output into two groups:**

```
✅ Valid records (N):
  ext-001 | 2026-07-01 | 82.4 kg | BMI 26.91 (overweight)
  ...

❌ Rejected records (M):
  ext-XXX | reason: weight 0 kg is not positive
  ext-YYY | reason: BMI category mismatch (source: normal, recalculated: overweight)
```

If M > 0, explain each rejection clearly. Do not proceed silently.

---

## Step 3: Map valid records to WeightEntry payloads

For each valid record, produce a payload in this exact shape:

```json
{
  "id": null,
  "userId": null,
  "date": "YYYY-MM-DD",
  "weightKg": 82.4,
  "bmi": 26.91,
  "bmiCategory": "overweight"
}
```

`id` and `userId` are always `null` in the output. The app repository must
generate a UUID v4 for `id` and set `userId` to the authenticated user's id
before executing the INSERT.

---

## Step 4: Present the final payloads

Show the complete array of WeightEntry payloads ready for INSERT, along with:
- Total count
- A clear synthetic data disclaimer if the source is sample data
- Instructions for what the repository must fill in (`id`, `userId`)

---

## Step 5: Repository INSERT pattern (reference)

Once the payloads are validated, this is the pattern for inserting them
in the target app (Drizzle ORM + SQLite):

```typescript
import { db } from '@/db';
import { weightEntry } from '@/db/schema';
import { randomUUID } from 'node:crypto';
import { FIXED_USER_ID } from '@/lib/constants';

async function insertImportedEntries(payloads) {
  await db.insert(weightEntry).values(
    payloads.map(p => ({
      ...p,
      id:        randomUUID(),
      userId:    FIXED_USER_ID,
      createdAt: new Date(),
      updatedAt: new Date(),
    }))
  );
}
```

See `references/validation-rules.md` for the full validation logic reference.
