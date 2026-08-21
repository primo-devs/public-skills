# Failure Modes Checklist

Scan this list when planning tests. For each mode, ask:

1. **Is it triggerable** in this code path?
2. **Do we own it** — or is it library/framework behavior?
3. **Can I name the code hook** — the specific branch or constraint?

If you can't answer all three, skip it.

---

## Inputs & Validation

| Mode                       | Look for                         |
| -------------------------- | -------------------------------- |
| Missing required fields    | nil/empty checks, required tags  |
| Empty collections          | len == 0 branches                |
| Malformed formats          | Date/money/ID parsing            |
| Invalid field combinations | Cross-field validation logic     |
| Boundary values            | Min/max checks, off-by-one risks |

---

## Data & Invariants

| Mode                     | Look for                                     |
| ------------------------ | -------------------------------------------- |
| Duplicate keys           | Upsert vs insert logic, unique constraints   |
| Idempotency violations   | Retry-safe operations, idempotency keys      |
| Missing related entities | Foreign key lookups, join assumptions        |
| Partial writes on error  | Multi-step mutations, transaction boundaries |
| Broken invariants        | Business rules that must hold post-operation |

---

## External Dependencies

| Mode                      | Look for                               |
| ------------------------- | -------------------------------------- |
| Timeout                   | Context handling, deadline propagation |
| Error responses           | Error branch coverage                  |
| Unexpected response shape | Unmarshaling, nil field access         |
| Partial success           | Batch operations, multi-call sequences |

---

## Concurrency & Ordering

| Mode                 | Look for                                 |
| -------------------- | ---------------------------------------- |
| Double processing    | At-least-once delivery, idempotency      |
| Lost updates         | Read-modify-write without locking        |
| Stale reads          | Cache invalidation, eventual consistency |
| Ordering assumptions | Event sequences, queue processing        |

---

## Error Handling & Safety

| Mode                  | Look for                                  |
| --------------------- | ----------------------------------------- |
| Wrong error surfaced  | Error wrapping, user-facing messages      |
| Error swallowed       | Ignored return values, empty catch blocks |
| Sensitive data leaked | Logging, error details, stack traces      |

---

## Mapping to Tests

Each selected failure mode needs a row in your test table:

| Test                         | Type        | What it catches        | Why realistic             | Code hook                      |
| ---------------------------- | ----------- | ---------------------- | ------------------------- | ------------------------------ |
| `TestCreateUser_EmptyName`   | Integration | Missing required field | Form submitted incomplete | `validate()` name check        |
| `TestCreateUser_DuplicateID` | Integration | Duplicate key rejected | Retry after timeout       | `insert()` unique constraint   |
| —                            | —           | DB timeout             | —                         | Skip: requires fault injection |
