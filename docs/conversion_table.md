# Type Conversion Reference

## SurrealDB → MessagePack (Query Results)

| SurrealDB `sql::Value` | MessagePack `rmpv::Value` | Notes |
|------------------------|---------------------------|-------|
| `None` | `Nil` | - |
| `Null` | `Nil` | - |
| `Bool(b)` | `Boolean(b)` | - |
| `Number::Int(i)` | `Integer(i)` | - |
| `Number::Float(f)` | `F64(f)` | - |
| `Number::Decimal(d)` | `F64(d.parse())` | Precision loss (15-17 digits) |
| `Strand(s)` | `String(s)` | - |
| `Duration(d)` | `String(d.to_string())` | Human-readable format |
| `Datetime(d)` | `String(rfc3339)` | `d'...'` wrapper removed |
| `Uuid(u)` | `String(u.to_string())` | Hyphenated format |
| `Thing(t)` | `String(id_only)` | Table prefix & brackets removed |
| `Array(a)` | `Array(...)` | Recursive conversion |
| `Object(o)` | `Map(...)` | Recursive conversion |
| `Bytes(b)` | `Binary(b)` | - |
| Others | `String(v.to_string())` | Fallback |

## MessagePack → SurrealDB (Parameters)

| MessagePack `rmpv::Value` | SurrealDB `sql::Value` | Notes |
|---------------------------|------------------------|-------|
| `Nil` | `None` | - |
| `Boolean(b)` | `Bool(b)` | - |
| `Integer(i)` (fits i64) | `Number::Int(i)` | - |
| `Integer(i)` (u64 > i64) | `Strand(i.to_string())` | Preserves value |
| `F64(f)` | `Number::Float(f)` | - |
| `F32(f)` | `Number::Float(f)` | - |
| `String(s)` | `Strand(s)` | See TypeHints below |
| `Binary(b)` | `Bytes(b)` | - |
| `Array(a)` | `Array(...)` | Recursive conversion |
| `Map(m)` | `Object(...)` | Recursive, TypeHints applied |
| `Ext(-1, bytes)` | `Datetime(...)` | MessagePack Timestamp |
| `Ext(other, bytes)` | `Bytes(bytes)` | Preserved as binary |

## TypeHints Application

TypeHints modify the String → sql::Value conversion:

### Datetime Hints

```
Hint: "created_at" or "datetime:created_at"

MessagePack String "2024-01-20T10:00:00Z"
    │
    └── Field name matches hint
        └── Parse as RFC3339 → sql::Datetime
```

### Record Hints

```
Hint: "record:user:owner_id"

MessagePack String "uuid-here"
    │
    └── Field name matches "owner_id"
        └── sql::Thing { tb: "user", id: "uuid-here" }
```

### Precedence

When a field has both datetime and record hints:
1. Record hint wins
2. Warning is logged (once per field)

## MessagePack Timestamp Extension

The MessagePack Timestamp Extension (type -1) is automatically converted:

| Format | Length | Structure |
|--------|--------|-----------|
| timestamp32 | 4 bytes | seconds (u32) |
| timestamp64 | 8 bytes | nanosec:30bits + seconds:34bits |
| timestamp96 | 12 bytes | nanosec (u32) + seconds (i64) |

## Thing ID Cleanup

SurrealDB 2.x wraps string IDs in mathematical angle brackets:

```
Original: user:⟨550e8400-e29b-41d4-a716-446655440000⟩
Cleaned:  550e8400-e29b-41d4-a716-446655440000
```

Characters removed: `⟨` (U+27E8) and `⟩` (U+27E9)

## Examples

### Query Result Conversion

```rust
// SurrealDB query result
let result = db.query("SELECT * FROM user:john").await?;

// Internal: sql::Value::Object {
//     "id": Thing { tb: "user", id: "john" },
//     "name": Strand("John Doe"),
//     "created_at": Datetime(2024-01-20T10:00:00Z),
//     "score": Number::Float(95.5),
// }

// After from_surreal_value():
// rmpv::Value::Map {
//     "id": String("john"),           // Thing → ID only
//     "name": String("John Doe"),
//     "created_at": String("2024-01-20T10:00:00Z"),  // RFC3339
//     "score": F64(95.5),
// }
```

### Parameter Conversion with Hints

```rust
let params = rmpv::Value::Map(vec![
    (String("owner_id"), String("user123")),
    (String("created_at"), String("2024-01-20T10:00:00Z")),
]);

let hints = TypeHints::parse(&[
    "record:user:owner_id",
    "created_at",
]);

let surreal_params = to_surreal_value_with_type_hints(params, &hints);

// Result: sql::Value::Object {
//     "owner_id": Thing { tb: "user", id: "user123" },
//     "created_at": Datetime(2024-01-20T10:00:00Z),
// }
```
