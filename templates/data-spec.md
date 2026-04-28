---
query: <one-line restatement>
generated_at: <iso8601>
budget_used: { reads: 0, bash: 0 }
source: <path | conn-string-redacted | url>
source_kind: <local-file|local-db|remote-db|warehouse|http|api>
row_count: <n or "unknown">
prep_applied: false
prep_path: ""
---

# Schema

| Column | Type | Nullable | Description |
|---|---|---|---|
| id | int | no | primary key |

# Sample Rows (5)

```
<verbatim, PII redacted>
```

# Source Access

- How to read: `<command or query>`
- Auth: `<env var or "none">`
- Refresh cadence: `<one-line>`

# Prep Notes (only if prep_applied)

- <transform> — <reason>

# Adapter Hints for Backend

- Suggested type mapping per column.
- Edge cases (nulls, encoding, timezones).
- Idempotency / dedup keys.

# Gaps

- <item> — <why>
