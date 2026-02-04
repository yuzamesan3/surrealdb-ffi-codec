# Integration Guide

This guide walks through integrating the SurrealDB FFI Codec into a new project.

## Prerequisites

- Rust toolchain (1.70+)
- FlatBuffers compiler (`flatc`)
- Target language SDK (Go 1.21+ / Swift 5.9+ / etc.)

## Step 1: Project Setup

### Cargo.toml

```toml
[package]
name = "your-project-db"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib", "staticlib"]

[dependencies]
# SurrealDB (choose appropriate feature)
surrealdb = { version = "2.0", default-features = false, features = ["kv-mem"] }
# For persistent storage, use: features = ["kv-rocksdb"]

# Serialization
flatbuffers = "24.3"
rmp-serde = "1.3"
rmpv = "1.3"

# Utilities
chrono = { version = "0.4", features = ["serde"] }
thiserror = "2.0"
log = "0.4"
```

### Directory Structure

```
your-project/
├── rust_lib/
│   ├── Cargo.toml
│   ├── build.rs              # Optional: auto-generate FlatBuffers
│   ├── schemas/
│   │   ├── request.fbs
│   │   └── response.fbs
│   └── src/
│       ├── lib.rs
│       ├── codec/
│       │   ├── mod.rs
│       │   ├── converter.rs
│       │   ├── msgpack.rs
│       │   └── generated/
│       │       ├── mod.rs
│       │       ├── request_generated.rs
│       │       └── response_generated.rs
│       ├── ffi/
│       │   ├── mod.rs
│       │   ├── wrapper.rs
│       │   └── error_codes.rs
│       └── db/                # Your database logic
│           ├── mod.rs
│           └── operations.rs
├── go_client/                 # Or swift_client, etc.
│   └── ...
└── scripts/
    └── build.sh
```

## Step 2: Define FlatBuffers Schemas

Copy templates from `resources/schemas/` and replace `{{NAMESPACE}}`:

```bash
# In schemas/request.fbs
namespace yourproject.ffi;
# ... rest of schema
```

Generate Rust code:

```bash
flatc --rust -o src/codec/generated schemas/request.fbs schemas/response.fbs
```

## Step 3: Implement Codec

Copy templates from `resources/codec/` and customize:

1. **mod.rs** - No changes needed
2. **converter.rs** - Add project-specific type conversions if needed
3. **msgpack.rs** - No changes needed

## Step 4: Implement FFI Layer

Copy templates from `resources/ffi/` and implement handlers.

### Runtime & DB Instance Management

FFI functions are called from non-async contexts. Use `OnceLock` to manage global singletons for the Tokio runtime and database connection:

```rust
// In wrapper.rs (or a dedicated globals.rs)

use std::sync::OnceLock;
use surrealdb::engine::local::{Db, Mem};  // or RocksDb for persistent
use surrealdb::Surreal;
use tokio::runtime::Runtime;

// ─────────────────────────────────────────────────────────────────
// Global Tokio Runtime (created once, reused for all FFI calls)
// ─────────────────────────────────────────────────────────────────
static RUNTIME: OnceLock<Runtime> = OnceLock::new();

fn get_runtime() -> &'static Runtime {
    RUNTIME.get_or_init(|| {
        Runtime::new().expect("Failed to create Tokio runtime")
    })
}

// ─────────────────────────────────────────────────────────────────
// Global DB Instance (initialized once with namespace/database)
// ─────────────────────────────────────────────────────────────────
static DB: OnceLock<Surreal<Db>> = OnceLock::new();

fn get_db_instance() -> &'static Surreal<Db> {
    DB.get_or_init(|| {
        get_runtime().block_on(async {
            let db = Surreal::new::<Mem>(()).await.expect("Failed to create DB");
            db.use_ns("app").use_db("main").await.expect("Failed to set namespace");
            db
        })
    })
}
```

### Handler Implementation

```rust
fn handle_select(request: Request) -> Vec<u8> {
    let payload = request.payload_as_select_request().unwrap();
    let table = payload.table_name();
    let record_id = payload.record_id();

    let db = get_db_instance();

    // Use the global runtime instead of creating a new one each call
    let result = get_runtime().block_on(async {
        if let Some(id) = record_id {
            db.select((table, id)).await
        } else {
            db.select(table).await
        }
    });

    match result {
        Ok(records) => {
            // Convert to MessagePack
            let msgpack = records
                .into_iter()
                .map(|r| converter::from_surreal_value(r.into_inner()))
                .collect::<Result<Vec<_>, _>>();

            match msgpack {
                Ok(values) => {
                    let bytes = msgpack::serialize_value(&Value::Array(values)).unwrap();
                    build_success_response(&bytes)
                }
                Err(e) => build_error_response(ErrorCode::TypeConversionError as i32, &e),
            }
        }
        Err(e) => build_error_response(ErrorCode::QueryError as i32, &e.to_string()),
    }
}
```

### RecordFieldHint の取り扱い

`record_fields` は `TypeHints` に合成して渡す。

- 例: `{ field: "owner_id", table_name: "user" }` → `record:user:owner_id`
- `datetime_fields` と合わせて `TypeHints::parse(&Vec<String>)` に渡す

## Step 5: Expose C API

In `lib.rs`:

```rust
pub mod codec;
pub mod ffi;
pub mod db;

pub use ffi::{execute_request, free_response_buffer, ResponseBuffer};
```

## Step 6: Build Native Library

```bash
# For macOS
cargo build --release
# Output: target/release/libyour_project_db.dylib

# For Linux
cargo build --release
# Output: target/release/libyour_project_db.so

# For iOS (static library)
cargo build --release --target aarch64-apple-ios
# Output: target/aarch64-apple-ios/release/libyour_project_db.a
```

## Step 7: Client Integration

### Go Client

```go
package db

/*
#cgo LDFLAGS: -L${SRCDIR}/../rust_lib/target/release -lyour_project_db
#include <stdint.h>

typedef struct {
    uint8_t* ptr;
    size_t len;
    size_t cap;
} ResponseBuffer;

extern ResponseBuffer execute_request(const uint8_t* request_ptr, size_t request_len);
extern void free_response_buffer(ResponseBuffer buf);
*/
import "C"
import (
    "unsafe"

    flatbuffers "github.com/google/flatbuffers/go"
    "github.com/vmihailenco/msgpack/v5"
)

func ExecuteSelect(table string, recordID *string) ([]map[string]interface{}, error) {
    // Build FlatBuffers request
    builder := flatbuffers.NewBuilder(256)
    tableOffset := builder.CreateString(table)

    var recordIDOffset flatbuffers.UOffsetT
    if recordID != nil {
        recordIDOffset = builder.CreateString(*recordID)
    }

    SelectRequestStart(builder)
    SelectRequestAddTableName(builder, tableOffset)
    if recordID != nil {
        SelectRequestAddRecordId(builder, recordIDOffset)
    }
    selectReq := SelectRequestEnd(builder)

    RequestStart(builder)
    RequestAddOperation(builder, OperationTypeQuerySelect)
    RequestAddPayloadType(builder, RequestPayloadSelectRequest)
    RequestAddPayload(builder, selectReq)
    request := RequestEnd(builder)
    builder.Finish(request)

    // Call FFI
    requestBytes := builder.FinishedBytes()
    response := C.execute_request(
        (*C.uint8_t)(unsafe.Pointer(&requestBytes[0])),
        C.size_t(len(requestBytes)),
    )
    defer C.free_response_buffer(response)

    // Parse response
    responseBytes := C.GoBytes(unsafe.Pointer(response.ptr), C.int(response.len))
    resp := GetRootAsResponse(responseBytes, 0)

    if resp.StatusCode() != 0 {
        var errPayload struct {
            Code    int32   `msgpack:"code"`
            Message string  `msgpack:"message"`
            Details *string `msgpack:"details"`
        }
        _ = msgpack.Unmarshal(resp.PayloadBytes(), &errPayload)
        return nil, fmt.Errorf("error %d: %s", errPayload.Code, errPayload.Message)
    }

    var result []map[string]interface{}
    if err := msgpack.Unmarshal(resp.PayloadBytes(), &result); err != nil {
        return nil, err
    }

    return result, nil
}
```

### Swift Client

```swift
import Foundation

// Bridge header or module.modulemap needed for C interop

struct ResponseBuffer {
    var ptr: UnsafeMutablePointer<UInt8>?
    var len: Int
    var cap: Int
}

@_silgen_name("execute_request")
func execute_request(_ request_ptr: UnsafePointer<UInt8>, _ request_len: Int) -> ResponseBuffer

@_silgen_name("free_response_buffer")
func free_response_buffer(_ buf: ResponseBuffer)

class DBClient {
    func executeSelect(table: String, recordID: String? = nil) throws -> [[String: Any]] {
        var builder = FlatBufferBuilder(initialSize: 256)

        let tableOffset = builder.create(string: table)
        let recordIDOffset = recordID.map { builder.create(string: $0) }

        let selectReq = SelectRequest.createSelectRequest(
            &builder,
            tableNameOffset: tableOffset,
            recordIdOffset: recordIDOffset ?? Offset()
        )

        let request = Request.createRequest(
            &builder,
            operation: .querySelect,
            payloadType: .selectRequest,
            payloadOffset: selectReq
        )
        builder.finish(offset: request)

        let requestBytes = builder.sizedByteArray

        let response = requestBytes.withUnsafeBufferPointer { ptr in
            execute_request(ptr.baseAddress!, requestBytes.count)
        }
        defer { free_response_buffer(response) }

        let responseData = Data(bytes: response.ptr!, count: response.len)
        let resp = Response.getRootAsResponse(bb: ByteBuffer(data: responseData))

        guard resp.statusCode == 0 else {
            struct ErrorPayload: Decodable {
                let code: Int32
                let message: String
                let details: String?
            }
            let err: ErrorPayload = try MessagePackDecoder().decode(from: resp.payloadArray!)
            throw DBError.queryFailed(code: err.code, message: err.message)
        }

        return try MessagePackDecoder().decode(from: resp.payloadArray!)
    }
}
```

## Performance Tips

1. **Reuse FlatBufferBuilder** - Don't create new builder for each request
2. **Pool ResponseBuffers** - Avoid frequent allocations on hot paths
3. **Batch Operations** - Use transactions for multiple writes
4. **Lazy Decoding** - Only decode MessagePack fields you need

## Troubleshooting

### Common Issues

1. **"Invalid FlatBuffers" error**
   - Check byte order (little-endian expected)
   - Verify schema version matches between Rust and client

2. **Type conversion errors**
   - Add appropriate TypeHints for datetime/record fields
   - Check RFC3339 format for datetime strings

3. **Memory leaks**
   - Always call `free_response_buffer` after processing
   - Use `defer` (Go) or `defer {}` (Swift) patterns
