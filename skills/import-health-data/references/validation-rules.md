# Validation Rules for Imported Health Records

These rules apply to every record before it is considered safe to insert.
They are derived from WHO physiological standards and the HealthTrack spec (RF-02, RF-03, RNF-01).

---

## Profile validation (run once per import batch)

| Check | Rule | On failure |
|---|---|---|
| Height present | `profile.heightCm` exists and is a number | Reject entire batch |
| Height positive | `heightCm > 0` | Reject entire batch |
| Height in range | `50 ≤ heightCm ≤ 300` | Reject entire batch |

---

## Per-record validation

### Weight
| Check | Rule | On failure |
|---|---|---|
| Positive | `weightKg > 0` | Reject record |
| Upper bound | `weightKg ≤ 700` | Reject record |
| Is a number | `typeof weightKg === 'number' && !isNaN(weightKg)` | Reject record |

### Date
| Check | Rule | On failure |
|---|---|---|
| Format | Matches `^\d{4}-\d{2}-\d{2}$` | Reject record |
| Not future | `date ≤ today + 1 day` (timezone tolerance) | Reject record |
| Valid calendar | Actual valid date (e.g. not `2026-02-30`) | Reject record |

### BMI consistency
| Check | Rule | On failure |
|---|---|---|
| Recalculate | `bmi = round(weightKg / (heightCm/100)², 2)` | — |
| Value match | `|source.bmi - recalculated| < 0.01` | Reject as inconsistent |
| Category match | `source.bmiCategory === classify(recalculated)` | Reject as inconsistent |
| Precision | `bmi === Math.round(bmi * 100) / 100` | Reject record |

### Category enum
| Check | Rule |
|---|---|
| Valid value | Must be one of: `underweight`, `normal`, `overweight`, `obese` |

---

## BMI classification thresholds

```
bmi < 18.5   → 'underweight'
bmi < 25.0   → 'normal'
bmi < 30.0   → 'overweight'
bmi ≥ 30.0   → 'obese'
```

---

## Rejection report format

Every rejected record must include:

```
id:     ext-XXX
date:   YYYY-MM-DD (or "missing" if absent)
reason: <specific rule that failed>
```

Example reasons:
- `weight 0 kg is not positive (must be > 0)`
- `date 2026-13-01 is not a valid calendar date`
- `BMI category mismatch: source says 'normal' but recalculated value 25.3 → 'overweight'`
- `height 45 cm is below minimum (50 cm)`
