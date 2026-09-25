---
inclusion: always
---

# Health Data Integration — Conventions & Guidelines

This steering file is loaded automatically when the `health-data-integration` Power
is active. It gives Kiro the domain context needed to map external health records
to internal schemas correctly.

---

## Core domain concepts

| Term | Definition |
|---|---|
| **External record** | A raw data point from a wearable or health export (Apple Health, Garmin, CSV). Contains at minimum: date, weight in kg. |
| **WeightEntry** | The internal schema record that the tracking app stores. Adds calculated BMI, BMI category, userId, and timestamps. |
| **Profile** | The user's static biometric data (height in cm, date of birth). Required to calculate BMI from weight. |
| **BMI** | Body Mass Index. Formula: `weight_kg / (height_cm / 100)²`. Rounded to 2 decimal places. |
| **Synthetic data** | Fictitious records used for demos and testing. Always marked with ⚠️ and never representing real measurements. |

---

## WeightEntry schema (canonical)

```
weight_entry table
─────────────────────────────────────────────
id           TEXT   PK, UUID v4 (generated on INSERT)
user_id      TEXT   FK → user_profile.id
date         TEXT   YYYY-MM-DD (ISO 8601, stored as TEXT in SQLite)
weight_kg    REAL   0 < x ≤ 700
bmi          REAL   calculated, 2 decimal places
bmi_category TEXT   'underweight' | 'normal' | 'overweight' | 'obese'
created_at   INT    Unix timestamp (auto)
updated_at   INT    Unix timestamp (auto)
```

`id` and `user_id` are always `null` in import payloads. The app repository
fills them before INSERT.

---

## BMI classification thresholds (WHO standard)

| Category | BMI range |
|---|---|
| underweight | < 18.5 |
| normal | 18.5 – 24.99 |
| overweight | 25.0 – 29.99 |
| obese | ≥ 30.0 |

---

## Validation rules for incoming records

These rules apply before any record is considered valid for import.
**Reject silently is never acceptable** — always report what was rejected and why.

### Weight
- `weightKg > 0` — must be positive
- `weightKg ≤ 700` — physiological upper bound

### Height (from source profile)
- `heightCm > 0` — must be positive
- `50 ≤ heightCm ≤ 300` — reasonable human range
- If height fails: reject ALL records from this source

### Date
- Format must be `YYYY-MM-DD`
- Must not be more than 1 day in the future (timezone tolerance)

### BMI consistency
- Recalculate BMI locally and compare with source value
- If recalculated category differs from source category: reject as inconsistent

---

## Mapping pattern: external record → WeightEntry payload

```typescript
// Pseudocode — adapt to the target ORM/language

function mapToWeightEntry(record, profileHeightCm) {
  validate(record, profileHeightCm);            // throws on invalid
  const bmi = calculateBMI(record.weightKg, profileHeightCm);
  return {
    id:          null,                          // filled by repository
    userId:      null,                          // filled by repository
    date:        record.date,                   // YYYY-MM-DD
    weightKg:    record.weightKg,
    bmi:         bmi.value,                     // 2 decimals
    bmiCategory: bmi.category,
  };
}
```

---

## MCP tools available (health-import server)

| Tool | Input | Output |
|---|---|---|
| `list_sample_records` | none | All records with pre-calculated BMI |
| `get_record_by_id` | `{ id: string }` | Single record detail |
| `import_records_to_weight_tracking` | `{ fromDate?, toDate?, ids? }` | WeightEntry payloads, filtered |

Reference the tool descriptions for exact input/output shapes.

---

## Privacy and data safety rules

1. **Never use real personal health data** in demos, tests, or examples. Use synthetic data only.
2. **Always label synthetic data** with a clear disclaimer in comments or `_meta` fields.
3. **Do not log or print** `userId` or any PII to stdout/stderr in the MCP server.
4. The MCP server communicates over stdio — its JSON-RPC channel must not include secrets.
