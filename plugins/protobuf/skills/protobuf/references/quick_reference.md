# Protocol Buffers Quick Reference

Fast lookup for common patterns and rules. For details, see [best_practices.md](best_practices.md).

## Field Numbering

| Range | Encoding | Use For |
|-------|----------|---------|
| 1-15 | 1 byte | Frequently-set fields |
| 16-2047 | 2 bytes | Standard fields |
| 2048-262143 | 3 bytes | Rarely-set fields |
| 19000-19999 | — | Reserved by protobuf (never use) |
| 536870912+ | — | Reserved (never use) |

**Rules:**
- Never reuse field numbers (reserve deleted ones)
- Avoid gaps in numbering
- Order in definition doesn't affect wire format

## Breaking Changes Checklist

### Safe Changes
- ✅ Add new fields
- ✅ Add new enum values
- ✅ Add new methods/services
- ✅ Add new messages
- ✅ Mark elements `deprecated`

### Wire-Safe but Breaks Generated Code
- ⚠️ Add `optional` to existing scalar fields
- ⚠️ Remove fields (reserve number + name)
- ⚠️ Remove enum values (reserve number + name)
- ⚠️ Move fields into a oneof
- ⚠️ Convert between compatible types (`int32` ↔ `int64`)

### Breaking (Requires New Version)
- ❌ Renumber fields
- ❌ Change field types incompatibly
- ❌ Rename fields (breaks JSON)
- ❌ Change RPC signatures
- ❌ Remove without reserving

## Scalar Type Selection

| Use Case | Type | Wire Encoding |
|----------|------|---------------|
| Regular integers | `int32`, `int64` | Varint (1-10 bytes) |
| Frequently negative | `sint32`, `sint64` | ZigZag varint (efficient for small absolute values) |
| Large values (>2²⁸) | `fixed32`, `fixed64` | Fixed 4/8 bytes |
| Arbitrary bytes | `bytes` | Length-prefixed |
| Text (UTF-8) | `string` | Length-prefixed |
| Boolean | `bool` | Varint (1 byte) |
| Decimals | `float`, `double` | Fixed 4/8 bytes |

**Tips:**
- Use signed types—many languages lack unsigned support
- Use `int32/int64` by default; switch to `sint*` if values are often negative
- Use `fixed*` only when values are consistently large

### Size Optimization

1. **Use field numbers 1-15** for frequently-set fields (1-byte tag)
2. **Use enums over strings** for fixed sets of values
3. **Consider `bytes` over `string`** if UTF-8 validation isn't needed
4. **Nested messages** add 2+ bytes overhead per instance

## Enum Template

```protobuf
enum Status {
  STATUS_UNSPECIFIED = 0;  // Required: zero = unset
  STATUS_ACTIVE = 1;
  STATUS_INACTIVE = 2;
}
```

**Rules:**
- First value must be `0` and `*_UNSPECIFIED`
- Prefix all values with enum name
- Never remove values—reserve them

## Message Template

```protobuf
message User {
  string id = 1;
  string email = 2;
  string display_name = 3;

  google.protobuf.Timestamp create_time = 10;
  google.protobuf.Timestamp update_time = 11;
}
```

## Service Template

```protobuf
service UserService {
  rpc GetUser(GetUserRequest) returns (User);
  rpc ListUsers(ListUsersRequest) returns (ListUsersResponse);
  rpc CreateUser(CreateUserRequest) returns (User);
  rpc UpdateUser(UpdateUserRequest) returns (User);
  rpc DeleteUser(DeleteUserRequest) returns (google.protobuf.Empty);
}
```

## Pagination Template

```protobuf
message ListUsersRequest {
  int32 page_size = 1;
  string page_token = 2;
}

message ListUsersResponse {
  repeated User users = 1;
  string next_page_token = 2;
}
```

## Field Mask Template

```protobuf
import "google/protobuf/field_mask.proto";

message UpdateUserRequest {
  User user = 1;
  google.protobuf.FieldMask update_mask = 2;
}
```

## Well-Known Types

| Import | Type | Use For |
|--------|------|---------|
| `google/protobuf/timestamp.proto` | `Timestamp` | Points in time |
| `google/protobuf/duration.proto` | `Duration` | Time spans |
| `google/protobuf/field_mask.proto` | `FieldMask` | Partial updates |
| `google/protobuf/empty.proto` | `Empty` | Empty responses |
| `google/protobuf/struct.proto` | `Struct` | Dynamic JSON |
| `google/protobuf/any.proto` | `Any` | Arbitrary messages |
| `google/protobuf/wrappers.proto` | `*Value` | Nullable scalars |

## Naming Conventions

| Element | Style | Example |
|---------|-------|---------|
| Files | `lower_snake_case.proto` | `user_service.proto` |
| Packages | `lower.dot.separated.v1` | `acme.user.v1` |
| Messages | `PascalCase` | `UserProfile` |
| Fields | `snake_case` | `display_name` |
| Services | `PascalCase` | `UserService` |
| RPCs | `PascalCase` | `GetUser` |
| Enums | `PascalCase` | `UserStatus` |
| Enum values | `UPPER_SNAKE_CASE` | `USER_STATUS_ACTIVE` |

## Common Buf Commands

```bash
buf lint                          # Lint protos
buf format -w                     # Format in place
buf breaking --against '.git#branch=main'  # Check breaking
buf generate                      # Generate code
buf dep update                    # Update dependencies
```

## Reserved Keywords

Avoid these in package names (cause issues in specific languages):

| Keyword | Language |
|---------|----------|
| `internal` | Go |
| `private` | Go |
| `class` | Python, Java, C# |
| `type` | TypeScript |
| `import` | Multiple |
| `package` | Multiple |
