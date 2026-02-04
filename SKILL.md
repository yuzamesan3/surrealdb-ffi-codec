---
name: surrealdb-ffi-codec
description: A codec implementation pattern for high‑efficiency FFI data exchange between SurrealDB Embedded and Go/Swift/other languages, eliminating JSON by using FlatBuffers + MessagePack. Used for Rust FFI layer construction, SurrealDB query result conversion, and binary serialization.
---

# SurrealDB FFI Codec

## Overview

An efficient binary codec pattern for accessing SurrealDB Embedded from other languages (Go, Swift, Kotlin, etc.) via FFI.

**Core principle**: eliminate JSON and optimize with a two‑layer structure: FlatBuffers (fixed schema) + MessagePack (dynamic schema).

## When to Use

- Embedding SurrealDB in Rust and accessing it from other languages via FFI
- Eliminating JSON parse/serialize overhead
- Handling dynamic query results while preserving type safety
- Requiring zero‑copy access through FlatBuffers

## Core Design Principle

| Data characteristics | Serialization | Examples |
|----------------------|---------------|----------|
| Fixed schema | FlatBuffers only | Error codes, status, operation types |
| Dynamic schema | FlatBuffers + embedded MessagePack | Query results, dynamic parameters |

## Conversion Flow (JSON Elimination)

```
surrealdb::Value (query result)
    | .into_inner()
    v
sql::Value (core layer)
    | from_surreal_value()
    v
rmpv::Value (MessagePack value)
    | serialize_to_msgpack()
    v
[u8] bytes (embedded in FlatBuffers payload)
```

## Type Mapping

| SurrealDB sql::Value | MessagePack rmpv::Value | Notes |
|----------------------|-------------------------|-------|
| None / Null | Nil | - |
| Bool | Boolean | - |
| Number::Int | Integer | - |
| Number::Float | F64 | - |
| Number::Decimal | F64 | Precision loss (15–17 digits) |
| Strand | String | - |
| Datetime | String | RFC3339, remove `d'...'` |
| Duration | String | - |
| Uuid | String | - |
| Thing | String | ID only, remove brackets |
| Array | Array | Recursive conversion |
| Object | Map | Recursive conversion |
| Bytes | Binary | - |
| Others | String | to_string() |

## Instructions

### Step 1: Confirm Project Layout

```
rust_lib/
├── Cargo.toml
├── schemas/
│   ├── request.fbs
│   └── response.fbs
├── src/
│   ├── lib.rs
│   ├── codec/
│   │   ├── mod.rs
│   │   ├── converter.rs
│   │   ├── msgpack.rs
│   │   └── generated/
│   │       ├── mod.rs
│   │       ├── request_generated.rs
│   │       └── response_generated.rs
│   └── ffi/
│       ├── mod.rs
│       ├── wrapper.rs
│       └── error_codes.rs
```

### Step 2: Implement Using Templates

Use templates under `resources/` and customize for your project.

1. **FlatBuffers schemas** (`resources/schemas/`)
   - Replace `{{NAMESPACE}}` with your project namespace
   - Define operation types

2. **Codec implementation** (`resources/codec/`)
   - Type conversion logic
   - Field‑level typing via `TypeHints`

3. **FFI wrapper** (`resources/ffi/`)
   - C ABI exports
   - Error codes

### Step 3: Generate FlatBuffers Code

```bash
flatc --rust -o src/codec/generated schemas/request.fbs schemas/response.fbs
```

### Step 4: Cargo.toml Dependencies

```toml
[dependencies]
surrealdb = { version = "2.x", features = ["kv-mem"] }  # or kv-rocksdb
flatbuffers = "24.x"
rmp-serde = "1.x"
rmpv = "1.x"
chrono = { version = "0.4", features = ["serde"] }
thiserror = "2.x"
log = "0.4"
```

## Guidelines

### Using TypeHints

Convert string parameters from clients into native SurrealDB types:

```rust
// Formats: "field_name" or "datetime:field_name" or "record:table:field"
let hints = TypeHints::parse(&[
    "created_at".to_string(),
    "record:user:owner_id".to_string(),
]);
let surreal_value = to_surreal_value_with_type_hints(msgpack_value, &hints);
```

### Decimal Precision Loss

`sql::Number::Decimal` is converted to f64, so precision beyond 15–17 digits is lost. If exact precision is required, treat it as a string.

### Thing ID Cleanup

SurrealDB 2.x wraps Thing IDs in angle brackets. Remove them during conversion:

```rust
// "user:⟨uuid-here⟩" -> "uuid-here"
let clean_id = id.trim_matches(|c| c == '\u{27E8}' || c == '\u{27E9}');
```

### Error Handling

- status_code = 0: success
- status_code = -1: Rust panic (fatal)
- status_code > 0: application error

### Applying RecordFieldHint

FlatBuffers `record_fields: [RecordFieldHint]` should be mapped into `TypeHints.record_fields`.

```
record_fields = [
  { field: "owner_id", table_name: "user" }
]
```

The above is equivalent to `record:user:owner_id`. Ensure client hints and Rust `TypeHints` are consistent.

### FFI Input Safety

- If `request_ptr` is null or `request_len == 0`, return `InvalidRequest` immediately.
- FFI callers must always invoke `free_response_buffer` and avoid double‑free.

### Error Payload Format

Use a structured MessagePack payload on error:

```
{ "code": i32, "message": String, "details": Option<String> }
```

Clients should ignore unknown fields for forward compatibility.

## Resources

- `resources/codec/` - Rust codec implementation templates
- `resources/ffi/` - FFI wrapper templates
- `resources/schemas/` - FlatBuffers schema templates
- `docs/` - detailed design documents

## See Also

- [FlatBuffers Documentation](https://flatbuffers.dev/)
- [MessagePack Specification](https://msgpack.org/)
- [SurrealDB SQL Types](https://surrealdb.com/docs/surrealql/datamodel)
