# Sample Health Data Schema

## File: `data/sample-health-data.json`

```json
{
  "_meta": {
    "note": "⚠️ SYNTHETIC DATA — fictitious records for demo purposes only. Not real health data."
  },
  "profile": {
    "heightCm": 175,
    "dateOfBirth": "1990-01-01"
  },
  "records": [
    {
      "id": "ext-001",
      "date": "YYYY-MM-DD",
      "weightKg": 80.0,
      "source": "apple_health",
      "notes": null
    }
  ]
}
```

## Field rules

| Field | Type | Constraint |
|---|---|---|
| `id` | string | Unique, prefixed `ext-`, e.g. `ext-001` |
| `date` | string | ISO 8601: `YYYY-MM-DD` |
| `weightKg` | number | `0 < x ≤ 700` |
| `source` | string | `"apple_health"` \| `"garmin"` \| `"manual"` |
| `notes` | string \| null | Optional free text |
| `profile.heightCm` | number | `50 ≤ x ≤ 300` |

## Synthetic data guidelines

- Use a realistic but invented weight trend (e.g. gradual loss of 0.3–0.5 kg/week)
- Span 2–3 months with 1–2 records per week
- Mix `apple_health` and `garmin` sources for variety
- Keep BMI in the `overweight` range for most records (realistic for a tracking app demo)
- Have at least one record crossing a category boundary to demonstrate classification
- Never use dates in the future beyond today + 1 day
