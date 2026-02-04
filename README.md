# SurrealDB FFI Codec

A Claude Code skill providing implementation patterns for high‑efficiency FFI data exchange between SurrealDB Embedded (Rust) and other languages (Go/Swift/Kotlin, etc.). It eliminates JSON by combining FlatBuffers (fixed schema) and MessagePack (dynamic schema).

## What is This?

This is a **Claude Code skill** that helps you implement efficient FFI layers for SurrealDB Embedded. When you ask Claude Code to help with SurrealDB FFI integration, it will use the templates and guidelines in this skill to generate appropriate code.

## When to Use

- Embedding SurrealDB in Rust and accessing it from other languages via FFI
- Eliminating JSON parse/serialize overhead
- Handling dynamic query results while preserving type safety
- Requiring zero‑copy access through FlatBuffers

## Architecture

```
SurrealDB sql::Value
  -> rmpv::Value (MessagePack)
  -> [u8] bytes embedded in FlatBuffers request/response
```

## Core Concepts

- **Two‑layer serialization**: FlatBuffers for fixed metadata, MessagePack for dynamic payloads.
- **TypeHints**: Field‑level hints to convert MessagePack strings into SurrealDB native types (datetime/record).
- **Structured errors**: MessagePack error payload with `{ code, message, details }`.

## Installation (Claude Code Skill)

Add this skill to your Claude Code configuration:

```bash
# Clone to your Claude skills directory
git clone https://github.com/yuzamesan3/surrealdb-ffi-codec.git ~/.claude/skills/surrealdb-ffi-codec
```

Or add it directly in Claude Code settings as a remote skill from GitHub.

## Usage

Once installed, simply ask Claude Code to help you with SurrealDB FFI implementation. For example:

- "Implement an FFI layer for SurrealDB"
- "Create a codec to access SurrealDB Embedded"
- "Write code to convert SurrealDB query results with FlatBuffers + MessagePack"

Claude Code will automatically reference this skill and use the templates under `resources/` to generate appropriate code for your project.

## Project Structure

```
resources/
  codec/       # Rust codec templates
  ffi/         # FFI wrapper templates
  schemas/     # FlatBuffers schema templates

docs/          # Design notes and integration guide
SKILL.md       # Skill specification (Claude Code reads this)
```

## Documentation

- [SKILL.md](SKILL.md) - Full skill specification and implementation guidelines
- [docs/design_principle.md](docs/design_principle.md) - Design principles
- [docs/integration_guide.md](docs/integration_guide.md) - Integration guide
- [docs/conversion_table.md](docs/conversion_table.md) - Type conversion table

## License

This project is licensed under the MIT License, see the LICENSE.txt file for details
