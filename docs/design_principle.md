# Design Principle: JSON Elimination via Two-Layer Serialization

## Problem Statement

Traditional FFI communication between Rust (SurrealDB) and other languages (Go, Swift) often relies on JSON as an intermediate format. This introduces several issues:

1. **Performance overhead**: JSON parsing is slow compared to binary formats
2. **Type ambiguity**: JSON has limited type support (no distinction between int/float, no datetime)
3. **Memory inefficiency**: JSON strings are larger than binary representations
4. **No schema validation**: JSON validation happens at runtime

## Solution: FlatBuffers + MessagePack

We employ a two-layer serialization strategy:

### Layer 1: FlatBuffers (Fixed Schema)

FlatBuffers handles:
- Operation types (SELECT, CREATE, UPDATE, DELETE, RAW_QUERY)
- Request/Response envelope
- Fixed metadata (table names, record IDs)
- Type hints for dynamic data conversion

**Benefits:**
- Zero-copy access on the receiver side
- Schema validation at compile time
- Efficient for fixed structures

### Layer 2: MessagePack (Dynamic Schema)

MessagePack handles:
- Query results (variable columns)
- User document data
- Query parameters

**Benefits:**
- Compact binary representation
- Self-describing format
- Preserves type information better than JSON

## Data Flow

```
┌────────────────────────────────────────────────────────────────┐
│                        Go/Swift Client                          │
├────────────────────────────────────────────────────────────────┤
│  1. Build FlatBuffers Request                                   │
│     - Operation type                                            │
│     - Table name / Query                                        │
│     - MessagePack-encoded params (if dynamic)                   │
│     - Type hints (datetime_fields, record_fields)               │
└─────────────────────────┬──────────────────────────────────────┘
                          │ [u8] FlatBuffers bytes
                          ▼
┌────────────────────────────────────────────────────────────────┐
│                      Rust FFI Layer                             │
├────────────────────────────────────────────────────────────────┤
│  2. Decode FlatBuffers (zero-copy)                              │
│  3. Decode MessagePack params → rmpv::Value                     │
│  4. Apply TypeHints → sql::Value                                │
│  5. Execute SurrealDB query                                     │
│  6. Convert sql::Value → rmpv::Value                            │
│  7. Encode as MessagePack                                       │
│  8. Wrap in FlatBuffers Response                                │
└─────────────────────────┬──────────────────────────────────────┘
                          │ [u8] FlatBuffers bytes
                          ▼
┌────────────────────────────────────────────────────────────────┐
│                        Go/Swift Client                          │
├────────────────────────────────────────────────────────────────┤
│  9. Decode FlatBuffers Response (zero-copy)                     │
│  10. Check status_code                                          │
│  11. Decode MessagePack payload                                 │
└────────────────────────────────────────────────────────────────┘
```

## Why Not Pure FlatBuffers?

FlatBuffers excels at fixed schemas but struggles with:
- Variable column queries (`SELECT * FROM table`)
- Nested JSON-like documents
- User-defined data structures

MessagePack fills this gap while remaining efficient.

## Why Not Pure MessagePack?

MessagePack lacks:
- Schema validation
- Union types
- Efficient enum representation

FlatBuffers provides these at the envelope level.

## TypeHints: Bridging the Gap

Since MessagePack strings can represent multiple SurrealDB types, we use TypeHints to guide conversion:

```
MessagePack String "2024-01-20T10:00:00Z"
    │
    ├── No hint → sql::Strand (string)
    │
    └── datetime_fields: ["created_at"]
        └── sql::Datetime
```

```
MessagePack String "uuid-here"
    │
    ├── No hint → sql::Strand (string)
    │
    └── record_fields: {"owner_id": "user"}
        └── sql::Thing(user:uuid-here)
```

## Precision Trade-offs

### Decimal → F64

SurrealDB's `Decimal` type has arbitrary precision, but MessagePack/FlatBuffers don't have a native decimal type. We convert to F64, accepting precision loss beyond 15-17 significant digits.

For applications requiring exact decimal precision:
1. Store as string in MessagePack
2. Parse on the client side

### Large Integers

`u64` values exceeding `i64::MAX` are converted to strings to preserve exact values.
