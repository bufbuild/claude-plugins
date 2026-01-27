# Protocol Buffers Examples

Complete service examples are in [assets/proto/example/v1/](../assets/proto/example/v1/).
Copy and adapt them as templates for your project.

## Service Template

### [book.proto](../assets/proto/example/v1/book.proto)

Entity definition demonstrating:
- Entity message with protovalidate constraints (uuid, patterns, ranges)
- BookRef oneof for flexible lookups (by ID or ISBN)
- Required oneof validation
- Enum with unspecified default
- Timestamp fields (create_time, update_time)

### [book_service.proto](../assets/proto/example/v1/book_service.proto)

A comprehensive CRUD service demonstrating:
- Standard and batch operations (Get, List, Create, Update, Delete, BatchGet, BatchCreate)
- BookRef pattern for lookups by ID or ISBN
- Protovalidate constraints (required, repeated bounds, enum.defined_only)
- Pagination with page_token and page_size
- Ordering with nested enum
- Field masks for partial updates
- Entity/service separation pattern

## Language Options

These examples omit language-specific file options (`go_package`, `java_package`, etc.).
Configure these via managed mode in `buf.gen.yaml`—see [buf_toolchain.md](buf_toolchain.md#managed-mode).
