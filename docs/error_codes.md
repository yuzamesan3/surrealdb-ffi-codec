# Error Codes

This document provides a comprehensive reference for all error codes used in the SurrealDB FFI Codec.

## Overview

The FFI layer uses standardized error codes to communicate success or failure across language boundaries. Error codes are returned in the `status_code` field of the FlatBuffers response.

### Code Ranges

| Range | Category | Description |
|-------|----------|-------------|
| 0 | Success | Operation completed successfully |
| -1 | Fatal | Rust panic (unrecoverable) |
| 100-199 | Request Errors | Invalid request format or parameters |
| 200-299 | Database Errors | SurrealDB operation failures |
| 300-399 | Serialization Errors | MessagePack encoding/decoding failures |
| 900-999 | Other | Reserved codes (e.g., not implemented) |

## Error Code Reference

### Success (0)

| Code | Name | Description |
|------|------|-------------|
| 0 | `Success` | Operation completed successfully |

**When it occurs**: The request was processed without errors.

**Client action**: Decode the `payload` field as MessagePack to retrieve the result data.

---

### Fatal Errors (-1)

| Code | Name | Description |
|------|------|-------------|
| -1 | `InternalError` | Rust panic occurred |

**When it occurs**:
- Unexpected null pointer dereference
- Array index out of bounds
- Assertion failure in Rust code
- Any unhandled panic in the FFI layer

**Client action**:
- Log the error with full payload message for debugging
- Do NOT retry - this indicates a bug in the Rust library
- Report the issue to the library maintainers

---

### Request Errors (1xx)

| Code | Name | Description |
|------|------|-------------|
| 100 | `InvalidRequest` | Invalid FlatBuffers request |
| 101 | `MissingField` | Missing required field |
| 102 | `InvalidValue` | Invalid field value |
| 103 | `UnsupportedOperation` | Unsupported operation type |

#### 100 - InvalidRequest

**When it occurs**:
- `request_ptr` is null
- `request_len` is 0
- FlatBuffers verification fails (corrupted or malformed data)
- Schema version mismatch between client and server

**Resolution**:
- Verify the request buffer is properly constructed
- Check byte order (FlatBuffers uses little-endian)
- Ensure schema versions match

#### 101 - MissingField

**When it occurs**:
- A required field (marked `required` in schema) is not set
- Example: `table_name` missing in SelectRequest

**Resolution**:
- Check the FlatBuffers schema for required fields
- Ensure all required fields are set before finishing the buffer

#### 102 - InvalidValue

**When it occurs**:
- Field value is syntactically valid but semantically incorrect
- Example: Empty string for `table_name`
- Example: Invalid RFC3339 format for datetime hint

**Resolution**:
- Validate field values before sending
- Check format requirements in documentation

#### 103 - UnsupportedOperation

**When it occurs**:
- `OperationType` enum value is not handled
- Using a newer operation type with an older library version

**Resolution**:
- Check supported operations in the library version
- Update the library if needed

---

### Database Errors (2xx)

| Code | Name | Description |
|------|------|-------------|
| 200 | `DatabaseConnection` | Database connection error |
| 201 | `QueryError` | Query execution error |
| 202 | `NotFound` | Record not found |
| 203 | `DuplicateRecord` | Duplicate record |
| 204 | `TransactionError` | Transaction error |

#### 200 - DatabaseConnection

**When it occurs**:
- SurrealDB instance not initialized
- Connection to embedded database failed
- RocksDB lock contention (if using kv-rocksdb)

**Resolution**:
- Ensure database is properly initialized before FFI calls
- Check file permissions for persistent storage
- Verify no other process holds the database lock

#### 201 - QueryError

**When it occurs**:
- SurrealQL syntax error
- Permission denied for the operation
- Invalid table or field reference
- Constraint violation

**Resolution**:
- Check the `message` field in error payload for details
- Validate SurrealQL syntax
- Verify permissions and schema

#### 202 - NotFound

**When it occurs**:
- SELECT/UPDATE/DELETE on non-existent record
- Table does not exist

**Resolution**:
- Verify the record ID exists before update/delete
- Check table name spelling
- Handle as expected condition (not always an error)

#### 203 - DuplicateRecord

**When it occurs**:
- CREATE with an ID that already exists
- Unique constraint violation

**Resolution**:
- Use UPSERT if you want to update existing records
- Generate unique IDs client-side
- Check for existing record before CREATE

#### 204 - TransactionError

**When it occurs**:
- Transaction timeout
- Deadlock detected
- Transaction already committed/rolled back

**Resolution**:
- Retry the transaction
- Reduce transaction scope
- Check for deadlock patterns in your code

---

### Serialization Errors (3xx)

| Code | Name | Description |
|------|------|-------------|
| 300 | `SerializationError` | MessagePack serialization error |
| 301 | `DeserializationError` | MessagePack deserialization error |
| 302 | `TypeConversionError` | Type conversion error |

#### 300 - SerializationError

**When it occurs**:
- Failed to encode SurrealDB result to MessagePack
- Unsupported value type in result

**Resolution**:
- Check the `details` field for the specific value that failed
- Report if this occurs with standard SurrealDB types

#### 301 - DeserializationError

**When it occurs**:
- Invalid MessagePack bytes in request `data` field
- Truncated or corrupted MessagePack data

**Resolution**:
- Verify MessagePack encoding on client-side
- Check for buffer truncation during FFI call

#### 302 - TypeConversionError

**When it occurs**:
- Failed to convert MessagePack value to SurrealDB type
- Invalid datetime string format (expected RFC3339)
- Invalid record ID format

**Resolution**:
- Use RFC3339 format for datetime: `2024-01-20T10:00:00Z`
- Ensure record IDs don't contain invalid characters
- Check TypeHints configuration

---

### Other Errors (9xx)

| Code | Name | Description |
|------|------|-------------|
| 999 | `NotImplemented` | Feature not implemented |

**When it occurs**:
- Calling an operation that exists in schema but not yet implemented
- Using a feature reserved for future use

**Resolution**:
- Check library documentation for supported features
- Wait for future release or contribute implementation

---

## Client Handling Examples

### Go

```go
package db

import (
    "errors"
    "fmt"

    "github.com/vmihailenco/msgpack/v5"
)

// ErrorCode represents FFI error codes
type ErrorCode int32

const (
    ErrSuccess            ErrorCode = 0
    ErrInternal           ErrorCode = -1
    ErrInvalidRequest     ErrorCode = 100
    ErrMissingField       ErrorCode = 101
    ErrInvalidValue       ErrorCode = 102
    ErrUnsupportedOp      ErrorCode = 103
    ErrDatabaseConnection ErrorCode = 200
    ErrQueryError         ErrorCode = 201
    ErrNotFound           ErrorCode = 202
    ErrDuplicateRecord    ErrorCode = 203
    ErrTransactionError   ErrorCode = 204
    ErrSerialization      ErrorCode = 300
    ErrDeserialization    ErrorCode = 301
    ErrTypeConversion     ErrorCode = 302
    ErrNotImplemented     ErrorCode = 999
)

// DBError represents a database error with code and message
type DBError struct {
    Code    ErrorCode
    Message string
    Details string
}

func (e *DBError) Error() string {
    if e.Details != "" {
        return fmt.Sprintf("[%d] %s: %s", e.Code, e.Message, e.Details)
    }
    return fmt.Sprintf("[%d] %s", e.Code, e.Message)
}

// IsRetryable returns true if the error might succeed on retry
func (e *DBError) IsRetryable() bool {
    switch e.Code {
    case ErrDatabaseConnection, ErrTransactionError:
        return true
    default:
        return false
    }
}

// IsNotFound returns true if the error indicates record not found
func (e *DBError) IsNotFound() bool {
    return e.Code == ErrNotFound
}

// ParseErrorPayload extracts error details from MessagePack payload
func ParseErrorPayload(payload []byte) *DBError {
    var errData struct {
        Code    int32   `msgpack:"code"`
        Message string  `msgpack:"message"`
        Details *string `msgpack:"details"`
    }

    if err := msgpack.Unmarshal(payload, &errData); err != nil {
        return &DBError{
            Code:    ErrInternal,
            Message: "Failed to parse error payload",
            Details: err.Error(),
        }
    }

    details := ""
    if errData.Details != nil {
        details = *errData.Details
    }

    return &DBError{
        Code:    ErrorCode(errData.Code),
        Message: errData.Message,
        Details: details,
    }
}

// HandleResponse processes FFI response and returns data or error
func HandleResponse(resp *Response) ([]byte, error) {
    if resp.StatusCode() == 0 {
        return resp.PayloadBytes(), nil
    }

    dbErr := ParseErrorPayload(resp.PayloadBytes())

    // Handle specific errors
    switch dbErr.Code {
    case ErrInternal:
        // Log panic for debugging - don't expose to user
        log.Printf("FATAL: Rust panic: %s", dbErr.Message)
        return nil, errors.New("internal error occurred")

    case ErrNotFound:
        // Often not a real error - return sentinel
        return nil, ErrRecordNotFound

    case ErrDatabaseConnection, ErrTransactionError:
        // Retryable - caller may retry
        return nil, dbErr

    default:
        return nil, dbErr
    }
}
```

### Swift

```swift
import Foundation

/// FFI Error codes
enum DBErrorCode: Int32 {
    case success = 0
    case internalError = -1

    // Request errors
    case invalidRequest = 100
    case missingField = 101
    case invalidValue = 102
    case unsupportedOperation = 103

    // Database errors
    case databaseConnection = 200
    case queryError = 201
    case notFound = 202
    case duplicateRecord = 203
    case transactionError = 204

    // Serialization errors
    case serializationError = 300
    case deserializationError = 301
    case typeConversionError = 302

    // Other
    case notImplemented = 999

    /// Unknown error code
    case unknown = -999

    init(rawValue: Int32) {
        switch rawValue {
        case 0: self = .success
        case -1: self = .internalError
        case 100: self = .invalidRequest
        case 101: self = .missingField
        case 102: self = .invalidValue
        case 103: self = .unsupportedOperation
        case 200: self = .databaseConnection
        case 201: self = .queryError
        case 202: self = .notFound
        case 203: self = .duplicateRecord
        case 204: self = .transactionError
        case 300: self = .serializationError
        case 301: self = .deserializationError
        case 302: self = .typeConversionError
        case 999: self = .notImplemented
        default: self = .unknown
        }
    }

    var isRetryable: Bool {
        switch self {
        case .databaseConnection, .transactionError:
            return true
        default:
            return false
        }
    }
}

/// Database error with structured information
struct DBError: Error, LocalizedError {
    let code: DBErrorCode
    let message: String
    let details: String?

    var errorDescription: String? {
        if let details = details {
            return "[\(code.rawValue)] \(message): \(details)"
        }
        return "[\(code.rawValue)] \(message)"
    }

    var isNotFound: Bool {
        code == .notFound
    }

    var isRetryable: Bool {
        code.isRetryable
    }
}

/// Error payload structure from MessagePack
struct ErrorPayload: Decodable {
    let code: Int32
    let message: String
    let details: String?
}

/// Parse error from FFI response
func parseDBError(from payload: Data) -> DBError {
    do {
        let errorPayload: ErrorPayload = try MessagePackDecoder().decode(from: payload)
        return DBError(
            code: DBErrorCode(rawValue: errorPayload.code),
            message: errorPayload.message,
            details: errorPayload.details
        )
    } catch {
        return DBError(
            code: .internalError,
            message: "Failed to parse error payload",
            details: error.localizedDescription
        )
    }
}

/// Handle FFI response with proper error handling
func handleResponse(_ response: Response) throws -> Data {
    guard response.statusCode == 0 else {
        let error = parseDBError(from: Data(response.payloadArray ?? []))

        switch error.code {
        case .internalError:
            // Log for debugging, don't expose details
            print("FATAL: Rust panic: \(error.message)")
            throw DBError(code: .internalError, message: "Internal error occurred", details: nil)

        case .notFound:
            // Throw specific error for not found
            throw error

        default:
            throw error
        }
    }

    guard let payload = response.payloadArray else {
        throw DBError(code: .internalError, message: "Empty payload", details: nil)
    }

    return Data(payload)
}

// MARK: - Usage Example

class DBClient {
    func select(table: String, id: String?) async throws -> [[String: Any]] {
        let response = try await executeRequest(/* ... */)

        do {
            let data = try handleResponse(response)
            return try MessagePackDecoder().decode(from: data)
        } catch let error as DBError {
            if error.isNotFound {
                return [] // Return empty array for not found
            }
            if error.isRetryable {
                // Retry logic here
            }
            throw error
        }
    }
}
```

## Error Payload Format

All error responses include a MessagePack-encoded payload with the following structure:

```
{
    "code": i32,       // Error code (same as status_code)
    "message": String, // Human-readable error message
    "details": String? // Optional additional details
}
```

Clients should:
1. Always check `status_code` first
2. Only parse payload as error if `status_code != 0`
3. Handle unknown fields gracefully (forward compatibility)
