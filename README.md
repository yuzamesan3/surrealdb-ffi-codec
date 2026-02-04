# SurrealDB FFI Codec

A reference implementation pattern for high‑efficiency FFI data exchange between SurrealDB Embedded and other languages (Go/Swift/Kotlin, etc.). It eliminates JSON by combining FlatBuffers (fixed schema) and MessagePack (dynamic schema).

## Why This Exists

- Avoid JSON parse/serialize overhead
- Preserve type information across FFI
- Keep zero‑copy envelopes with FlatBuffers
- Support dynamic query results with MessagePack

## Architecture

```
SurrealDB sql::Value
  -> rmpv::Value (MessagePack)
  -> [u8] bytes embedded in FlatBuffers request/response
```

## Project Structure

```
resources/
  codec/       # Rust codec templates
  ffi/         # FFI wrapper templates
  schemas/     # FlatBuffers schema templates

docs/          # Design notes and integration guide
SKILL.md       # Skill specification
```

## Core Concepts

- **Two‑layer serialization**: FlatBuffers for fixed metadata, MessagePack for dynamic payloads.
- **TypeHints**: Field‑level hints to convert MessagePack strings into SurrealDB native types (datetime/record).
- **Structured errors**: MessagePack error payload with `{ code, message, details }`.

## Getting Started

1. Copy templates from `resources/` into your Rust project.
2. Replace `{{NAMESPACE}}` in the FlatBuffers schemas.
3. Generate Rust bindings with `flatc`.
4. Implement the FFI handlers and database wiring.

## Documentation

- [docs/design_principle.md](docs/design_principle.md)
- [docs/integration_guide.md](docs/integration_guide.md)
- [docs/conversion_table.md](docs/conversion_table.md)

## License

This project is licensed under the MIT License, see the LICENSE.txt file for details
