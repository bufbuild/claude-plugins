# Protocol Buffers Best Practices

Universal best practices for designing `.proto` files.
For buf CLI configuration, see [buf_toolchain.md](buf_toolchain.md).

**Validation:** Use [protovalidate](protovalidate.md) on every field.
Protobuf defines structure; protovalidate defines what valid data looks like.
Together they make the `.proto` file the single source of truth for your API contract.

## Contents

- [File Structure](#file-structure)
- [Package Naming](#package-naming)
- [Message Design](#message-design)
- [Field Numbering](#field-numbering)
- [Scalar Type Selection](#scalar-type-selection)
- [Field Presence](#field-presence)
- [Enums](#enums)
- [Oneof](#oneof)
- [Maps](#maps)
- [Well-Known Types](#well-known-types)
- [Services and RPCs](#services-and-rpcs)
- [Schema Evolution](#schema-evolution)
- [Documentation](#documentation)
- [Common Patterns](#common-patterns)
- [API Design Principles](#api-design-principles)
- [Commit Generated Code](#commit-generated-code)

---

## File Structure

- Use `lower_snake_case.proto`
- Directory structure should match package: `acme/user/v1/user.proto` for `acme.user.v1`
- Keep files focused—large files with many unrelated messages cause dependency bloat
- Line length: soft limit 80 characters, hard limit 120 characters
- 2-space indentation
- Use `//` comments over `/* */`

### Element Ordering Within Files

Order elements consistently within each proto file:
1. License header (if applicable)
2. File-level documentation (if applicable)
3. Syntax declaration
4. Package declaration
5. Import statements
6. Options
7. Services (if any)
8. Messages (with related Ref messages immediately after their entity)
9. Enums (standalone, not nested in messages)

This ordering ensures services (the primary API contract) appear first, and keeps
reference messages close to their entities.

## Package Naming

- Use dot-delimited lowercase: `company.product.domain.version`
- Always include version suffix: `acme.user.v1`, `acme.billing.v2alpha1`
- Avoid language keywords in package names (e.g., `internal` blocks Go imports)
- Files in the same package must be in the same directory

```protobuf
syntax = "proto3";

package acme.user.v1;
```

## Message Design

### Naming

- Use `PascalCase`: `UserProfile`, `OrderItem`
- Use singular nouns for non-repeated entities
- Be specific: `CreateUserRequest` not `Request`
- Treat abbreviations as single words: `GetDnsRequest` not `GetDNSRequest`

### Field Naming

- Use `snake_case`: `user_id`, `created_at`
- Avoid abbreviations: `message` not `msg`
- Pluralize repeated fields: `repeated User users`
- Add [protovalidate](protovalidate.md) constraints to every field—format validators, length/range bounds, enum constraints, and required markers are all part of the field definition

### Nesting

Define messages and enums at the top level unless exclusively used by the parent.
Exception: request-scoped types like `ListBooksRequest.OrderBy` that have no external meaning.

## Field Numbering

### Wire Encoding Efficiency

- **1-15**: Use for frequently-set fields (1 byte tag encoding)
- **16-2047**: Standard fields (2 byte tag encoding)
- **19000-19999**: Reserved by Protocol Buffers—never use
- **536870912-max**: Reserved—never use

### Never Reuse Field Numbers

When a field is removed, reserve both the number and name:

```protobuf
message User {
  reserved 4, 8 to 10;
  reserved "legacy_field", "old_status";

  string id = 1;
  string email = 2;
  string name = 3;
  string display_name = 5;
}
```

Reusing field numbers causes data corruption when decoding messages serialized with old schemas.

## Scalar Type Selection

| Use Case | Type | Notes |
|----------|------|-------|
| Regular integers | `int32`, `int64` | Default choice |
| Likely negative values | `sint32`, `sint64` | More efficient encoding |
| Large positive values (>2^28) | `fixed32`, `fixed64` | Constant 4/8 bytes |
| Arbitrary bytes | `bytes` | Not for text |
| Text | `string` | Must be UTF-8 |
| Signed integers | Use signed types | Many languages lack unsigned support |

### Unsigned vs Signed Types

Prefer signed types (`int32`/`int64`) for public APIs since Java and JavaScript lack unsigned support.
For internal APIs, unsigned types are fine.
When using signed types for non-negative values, add validation: `(buf.validate.field).int32.gte = 0`

## Field Presence

Proto3 has three presence modes:

1. **Implicit presence** (default for scalars): Cannot distinguish unset from zero value
2. **Explicit presence** (`optional` keyword): Can detect if field was set
3. **Message fields**: Always have presence semantics

```protobuf
message Example {
  string name = 1;              // Implicit: "" means unset or empty
  optional string nickname = 2; // Explicit: can detect unset vs ""
  Address address = 3;          // Message: has presence (null vs empty)
}
```

Use `optional` when distinguishing "not provided" from "provided as empty/zero" matters.

## Enums

```protobuf
enum UserStatus {
  USER_STATUS_UNSPECIFIED = 0;
  USER_STATUS_ACTIVE = 1;
  USER_STATUS_INACTIVE = 2;
  USER_STATUS_SUSPENDED = 3;
}
```

### Rules

- First value must be zero and named `*_UNSPECIFIED` or `*_UNKNOWN`
- The zero value should have no semantic meaning—it indicates "not set"
- Prefix all values with the enum name in `UPPER_SNAKE_CASE` to avoid collisions
- Never remove enum values—reserve them instead
- Do not use `allow_alias`

### Deprecation

```protobuf
enum Status {
  STATUS_UNSPECIFIED = 0;
  STATUS_ACTIVE = 1;
  STATUS_LEGACY = 2 [deprecated = true];

  reserved 3;
  reserved "STATUS_REMOVED";
}
```

### Enum Value Documentation

Place comments above each enum value, not inline (inline comments often don't appear in generated docs).
Avoid grouping comments above multiple values; comments only attach to the first value.

### Required Enum Fields

For enum fields that must have a meaningful value (not UNSPECIFIED), always use **both** `not_in = 0` and `defined_only = true`:

```protobuf
Status status = 1 [
  (buf.validate.field).enum.not_in = 0,        // rejects UNSPECIFIED
  (buf.validate.field).enum.defined_only = true // rejects unknown values
];
```

Using only `defined_only` allows `UNSPECIFIED` through.
Using only `not_in = 0` allows unknown enum values through.

**Exception:** Optional enum fields where the zero value means "not set" or "no preference" should use only `defined_only = true`.
This is common for enum fields in List requests where the zero value means "return all."

See [protovalidate.md](protovalidate.md#enum-validation-patterns) for the full pattern table.

## Oneof

Use `oneof` when exactly one of several fields should be set:

```protobuf
message SearchQuery {
  oneof query {
    option (buf.validate.oneof).required = true;
    string text = 1;
    int64 id = 2;
    EmailFilter email = 3;
  }
}
```

**Validation:** If the oneof represents a required choice (lookups, mutually exclusive options), always add `(buf.validate.oneof).required = true`.
See [protovalidate.md](protovalidate.md#oneof-rules) for when to use `required` vs. omit it.

**Behavior:** Setting any member clears all others. Cannot distinguish "not set" from "set to removed field" across versions.

**Evolution:** Adding fields to existing oneof is safe. Moving existing fields into a oneof or removing fields is unsafe.

## Maps

```protobuf
message Project {
  map<string, string> labels = 1;
  map<string, User> members = 2;
}
```

### Map Constraints

- Keys: integral or string types only (no floats, bytes, enums, or messages)
- Values: any type except maps
- Iteration order is undefined

## Well-Known Types

Prefer standard types from `google/protobuf`:

| Type | Use For |
|------|---------|
| `google.protobuf.Timestamp` | Points in time |
| `google.protobuf.Duration` | Time spans |
| `google.protobuf.FieldMask` | Partial updates |
| `google.protobuf.Struct` | Dynamic JSON-like data |
| `google.protobuf.Any` | Arbitrary message types |

**Note:** Prefer custom empty messages over `google.protobuf.Empty` for extensibility.

## Services and RPCs

### Naming

- Service names: `PascalCase`, typically with `Service` suffix
- RPC names: `PascalCase` verbs: `GetUser`, `CreateOrder`, `ListProducts`
- Be specific: `ActivateUser` not `SetUserStatus`

### Request/Response Messages

- Name as `MethodNameRequest` and `MethodNameResponse`
- Each RPC should have unique request/response types (enables future evolution)
- Avoid reusing request types across RPCs
- Every request field should have protovalidate constraints—requests are the primary system boundary where validation matters most

```protobuf
service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
  rpc ListUsers(ListUsersRequest) returns (ListUsersResponse);
  rpc CreateUser(CreateUserRequest) returns (CreateUserResponse);
}
```

## Schema Evolution

### Adding Fields

Add new fields with the next available field number and appropriate protovalidate constraints:

```protobuf
message User {
  string id = 1 [(buf.validate.field).string.uuid = true];
  string email = 2 [(buf.validate.field).string.email = true];
  string name = 3 [(buf.validate.field).string.min_len = 1];
  string phone = 4;  // New field - add validation constraints
}
```

Use numbers 1-15 for common fields (1-byte encoding).
Avoid gaps in field numbers—gaps make it unclear whether a field was intentionally skipped or removed without reservation.

### Deprecating and Removing Fields

Mark fields as deprecated to warn consumers:

```protobuf
message User {
  string id = 1;
  // Deprecated: Use display_name instead.
  string name = 2 [deprecated = true];
  string display_name = 3;
}
```

Keep deprecated fields indefinitely for published APIs.
For internal APIs, you may eventually remove and reserve after migration:

```protobuf
message User {
  reserved 2;
  reserved "name";

  string id = 1;
  string display_name = 3;
}
```

### Replacing Fields with Different Types

Add a new field rather than modifying the existing one:

```protobuf
message User {
  string role = 3 [deprecated = true];  // Keep for compatibility
  UserRole user_role = 4;               // New typed field
}

enum UserRole {
  USER_ROLE_UNSPECIFIED = 0;
  USER_ROLE_ADMIN = 1;
  USER_ROLE_MEMBER = 2;
}
```

### Versioning Strategy

**Evolve within a version** for backwards-compatible changes (adding fields, enum values, methods; deprecating elements).

**Create new version** (`v1` -> `v2`) for breaking changes (type changes, semantic changes, restructuring).

### Testing Schema Evolution

Use `buf breaking --against '.git#branch=main'` to catch unintentional breaking changes.

## Documentation

Document all public APIs:

```protobuf
// UserService provides user management operations.
service UserService {
  // GetUser retrieves a user by their unique identifier.
  //
  // Returns NOT_FOUND if the user does not exist.
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
}

// User represents a registered user account.
message User {
  // Unique identifier for the user. Format: UUID v4
  string id = 1;
}
```

- Use complete sentences with proper punctuation
- Document constraints, formats, and valid ranges
- Note error conditions on RPCs
- Skip comments on request/response messages (names are self-documenting); document fields within them

### Terminology Consistency

Pick one form of a term and use it everywhere.
Mixed terminology confuses both users and implementers.

Common examples:
- "shortname" vs "short name" — pick one
- "ID" vs "id" vs "Id" in comments — use "ID"
- Entity names in comments should be capitalized consistently ("Source" not "source" when referring to the entity)

### Grammar

Watch for article/vowel mismatches, especially with entity names:
- "an Environment" not "a Environment"
- "an Image" not "a Image"
- "a Source" not "an Source"

### Cross-Reference Consistency

When the same identifier appears in multiple messages, all validation constraints (`max_len`, `min_len`, `pattern`) must be identical.
For example, if `Project.name` allows 64 characters, then `TaskName.project` (which references the same project name) must also allow exactly 64 characters.
See [protovalidate.md](protovalidate.md#cross-reference-consistency) for examples.

## Common Patterns

### Pagination and List Requests

```protobuf
message ListUsersRequest {
  // Nested Order enum scoped to this request.
  enum Order {
    ORDER_UNSPECIFIED = 0;
    ORDER_CREATE_TIME_DESC = 1;
    ORDER_CREATE_TIME_ASC = 2;
    ORDER_UPDATE_TIME_DESC = 3;
    ORDER_UPDATE_TIME_ASC = 4;
  }

  uint32 page_size = 1 [(buf.validate.field).uint32.lte = 250];
  string page_token = 2 [(buf.validate.field).string.max_len = 4096];
  // The order to return results.
  //
  // If not specified, defaults to ORDER_CREATE_TIME_DESC.
  Order order = 3 [(buf.validate.field).enum.defined_only = true];
}

message ListUsersResponse {
  string next_page_token = 1 [(buf.validate.field).string.max_len = 4096];
  repeated User users = 2 [(buf.validate.field).repeated.max_items = 250];
}
```

### Partial Updates with Field Masks

```protobuf
import "google/protobuf/field_mask.proto";

message UpdateUserRequest {
  User user = 1;
  google.protobuf.FieldMask update_mask = 2;
}
```

### Resource Metadata

```protobuf
message User {
  string id = 1;
  string email = 2;

  google.protobuf.Timestamp create_time = 10;
  google.protobuf.Timestamp update_time = 11;
}
```

### Soft Delete

```protobuf
message User {
  string id = 1;
  // Set when the user is soft-deleted. Null if active.
  google.protobuf.Timestamp delete_time = 20;
}
```

### Embedding vs Reference

Embed when data is small and always needed together (`Address shipping_address = 1`).
Reference by ID when data is large, optional, or separately managed (`string user_id = 1`).

### Reference Messages (Ref Pattern)

Use `*Ref` messages for flexible entity lookups.
Place immediately after the entity in the same file.
Each oneof field must be a unique identifier.

```protobuf
message UserRef {
  oneof value {
    option (buf.validate.oneof).required = true;
    string id = 1 [(buf.validate.field).string.uuid = true];
    string email = 2 [(buf.validate.field).string.email = true];
  }
}
```

### Create and Update Request Patterns

**Flat Fields (Recommended):** Request contains only user-modifiable fields directly.
Clear validation, but duplicates fields between entity and request.

```protobuf
message CreateUserRequest {
  string name = 1 [(buf.validate.field).required = true];
  string email = 2 [(buf.validate.field).required = true];
}

message UpdateUserRequest {
  string id = 1 [(buf.validate.field).string.uuid = true];
  optional string name = 2;
  optional string email = 3;
}
```

**Entity with FieldMask ([AIP-134](https://google.aip.dev/134)):** Reuses entity definition.
Skip validation on entity field: `User user = 1 [(buf.validate.field).ignore = IGNORE_ALWAYS];`

## API Design Principles

### Enums Over Booleans

Use enums instead of booleans when state might expand:

```protobuf
// Instead of: bool active = 1;
message User {
  UserStatus status = 1;
}

enum UserStatus {
  USER_STATUS_UNSPECIFIED = 0;
  USER_STATUS_ACTIVE = 1;
  USER_STATUS_INACTIVE = 2;
  USER_STATUS_SUSPENDED = 3;
}
```

### Oneofs for Mutually Exclusive State

Use `oneof` to model mutually exclusive states explicitly:

```protobuf
message PaymentMethod {
  oneof method {
    CreditCard credit_card = 1;
    BankAccount bank_account = 2;
    PayPalAccount paypal = 3;
  }
}
```

This makes the API self-documenting and prevents invalid states at the schema level.

### Format Validation Implies Presence

Many format validations reject empty strings, making `required = true` redundant.
Examples include `uuid`, `email`, `uri`, `hostname`, `ip`, and others.

```protobuf
// Sufficient - format validation handles presence
string id = 1 [(buf.validate.field).string.uuid = true];
string email = 2 [(buf.validate.field).string.email = true];
```

## Commit Generated Code

Commit generated proto code to version control.
This enables direct imports without running `buf generate`, reproducible builds, and immediate IDE support.
Do NOT add generated proto code to `.gitignore`.
