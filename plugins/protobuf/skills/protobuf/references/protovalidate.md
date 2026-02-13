# Protovalidate Reference

Protovalidate declares validation constraints directly in `.proto` files, making the schema the single source of truth for both structure and validation.
Rules are field annotations enforced at runtime via generated code and CEL expressions.

**Documentation:** [protovalidate.com](https://protovalidate.com)

## Contents

- [Setup](#setup)
- [Core Concepts](#core-concepts)
- [String Rules](#string-rules)
- [Common String Patterns](#common-string-patterns)
- [Numeric Rules](#numeric-rules)
- [Bytes Rules](#bytes-rules)
- [Enum Rules](#enum-rules)
- [Repeated Field Rules](#repeated-field-rules)
- [Map Rules](#map-rules)
- [Oneof Rules](#oneof-rules)
- [Timestamp Rules](#timestamp-rules)
- [Duration Rules](#duration-rules)
- [Custom CEL Rules](#custom-cel-rules)
- [Common Patterns](#common-patterns)
- [Runtime Validation](#runtime-validation)

---

## Setup

Add protovalidate as a dependency:

```yaml
# buf.yaml
version: v2
modules:
  - path: proto
deps:
  - buf.build/bufbuild/protovalidate
```

Run `buf dep update`, then import in proto files:

```protobuf
import "buf/validate/validate.proto";
```

## Core Concepts

### Presence and Validation

- **Explicit presence** (proto3 `optional`, messages, oneofs): Unset fields skip validation unless marked `required`
- **Implicit presence** (proto3 scalars without `optional`): Fields always validate against their current value
- **Nested validation**: Child messages are automatically validated; parent fails if any child fails

### The `required` Rule

`required` means a field must be **set** (present), not that it must have a non-default value:

```protobuf
// With explicit presence: fails if field isn't set
optional string name = 1 [(buf.validate.field).required = true];

// Without explicit presence: fails if field equals default value
string name = 1 [(buf.validate.field).required = true];  // fails on ""
```

Empty strings and zero values pass `required` if the field is set/present.

## String Rules

```protobuf
message Example {
  // Length constraints (character count)
  string username = 1 [
    (buf.validate.field).string.min_len = 3,
    (buf.validate.field).string.max_len = 32
  ];

  // Byte length constraints
  string bio = 2 [
    (buf.validate.field).string.min_bytes = 1,
    (buf.validate.field).string.max_bytes = 1024
  ];

  // Exact length
  string country_code = 3 [(buf.validate.field).string.len = 2];

  // Pattern matching (RE2 regex)
  string slug = 4 [(buf.validate.field).string.pattern = "^[a-z][a-z0-9-]*$"];

  // Substring matching
  string path = 5 [(buf.validate.field).string.prefix = "/api/"];
  string filename = 6 [(buf.validate.field).string.suffix = ".proto"];
  string description = 7 [(buf.validate.field).string.contains = "important"];
  string safe_text = 8 [(buf.validate.field).string.not_contains = "<script>"];

  // Format validators
  string email = 9 [(buf.validate.field).string.email = true];
  string website = 10 [(buf.validate.field).string.uri = true];
  string id = 11 [(buf.validate.field).string.uuid = true];
  string compact_id = 12 [(buf.validate.field).string.tuuid = true];  // UUID without dashes

  // Network formats
  string host = 13 [(buf.validate.field).string.hostname = true];
  string ip = 14 [(buf.validate.field).string.ip = true];
  string address = 15 [(buf.validate.field).string.address = true];  // hostname or IP
  string endpoint = 16 [(buf.validate.field).string.host_and_port = true];

  // Value sets
  string status = 17 [(buf.validate.field).string = {in: ["active", "inactive"]}];
  string role = 18 [(buf.validate.field).string = {not_in: ["admin", "root"]}];
}
```

## Common String Patterns

Prefer well-known format validators (`uuid`, `email`, `uri`, `hostname`, `ip`, `address`, `host_and_port`) before reaching for `pattern`.
When no well-known constraint fits, use `pattern` for name/identifier fields:

| Field Type | Pattern |
|-----------|---------|
| Lowercase with hyphens | `^[a-z0-9][a-z0-9-]*[a-z0-9]$` |
| Versioned label | `^[a-z0-9]([a-z0-9._-]*[a-z0-9])?$` |
| Programming identifier | `^[_a-zA-Z][_a-zA-Z0-9]*$` |

Consider adding `min_len` and `max_len` constraints where appropriate.

## Numeric Rules

All numeric types (`int32`, `int64`, `uint32`, `uint64`, `sint32`, `sint64`, `fixed32`, `fixed64`, `sfixed32`, `sfixed64`, `float`, `double`) support:

```protobuf
message Pagination {
  // Comparison operators
  uint32 page_size = 1 [
    (buf.validate.field).uint32.gt = 0,
    (buf.validate.field).uint32.lte = 100
  ];

  int32 offset = 2 [
    (buf.validate.field).int32.gte = 0,
    (buf.validate.field).int32.lte = 10000
  ];

  // Value sets
  int32 version = 3 [(buf.validate.field).int32 = {in: [1, 2, 3]}];
  int32 port = 4 [(buf.validate.field).int32 = {not_in: [0, 22, 80, 443]}];

  // Exact value
  int32 api_version = 5 [(buf.validate.field).int32.const = 1];
}

message Coordinates {
  double latitude = 1 [
    (buf.validate.field).double.gte = -90,
    (buf.validate.field).double.lte = 90
  ];

  double longitude = 2 [
    (buf.validate.field).double.gte = -180,
    (buf.validate.field).double.lte = 180
  ];

  // Prevent infinity and NaN
  double distance = 3 [(buf.validate.field).double.finite = true];
}
```

## Bytes Rules

```protobuf
message Upload {
  bytes content = 1 [
    (buf.validate.field).bytes.min_len = 1,
    (buf.validate.field).bytes.max_len = 10485760  // 10MB
  ];

  // Magic bytes check
  bytes png_image = 2 [(buf.validate.field).bytes.prefix = "\x89PNG"];

  // IP address as bytes
  bytes ip_addr = 3 [(buf.validate.field).bytes.ip = true];
}
```

## Enum Rules

```protobuf
enum Status {
  STATUS_UNSPECIFIED = 0;
  STATUS_ACTIVE = 1;
  STATUS_INACTIVE = 2;
}

message Resource {
  // Must be a defined, non-UNSPECIFIED value
  Status status = 1 [
    (buf.validate.field).enum.not_in = 0,
    (buf.validate.field).enum.defined_only = true
  ];

  // Specific allowed values
  Status allowed = 2 [(buf.validate.field).enum = {in: [1, 2]}];

  // Excluded values
  Status restricted = 3 [(buf.validate.field).enum = {not_in: [0]}];

  // Exact value
  Status required_status = 4 [(buf.validate.field).enum.const = 1];
}
```

### Enum Validation Patterns

| Context | Constraints |
|---------|-------------|
| Required enum | `not_in = 0` + `defined_only = true` |
| Optional enum (zero = "not set") | `defined_only = true` only |
| Repeated enum items | `.repeated.items.enum.not_in = 0` + `.items.enum.defined_only = true` |

Required fields need both: `defined_only` alone allows `UNSPECIFIED` through, `not_in = 0` alone allows unknown values through.

## Repeated Field Rules

```protobuf
message BatchRequest {
  // Size constraints
  repeated string ids = 1 [
    (buf.validate.field).repeated.min_items = 1,
    (buf.validate.field).repeated.max_items = 250
  ];

  // Unique items (scalars and enums only)
  repeated string tags = 2 [(buf.validate.field).repeated.unique = true];

  // Validate each item
  repeated string emails = 3 [
    (buf.validate.field).repeated.items.string.email = true
  ];
}
```

## Map Rules

```protobuf
message Config {
  // Entry count constraints
  map<string, string> labels = 1 [
    (buf.validate.field).map.min_pairs = 1,
    (buf.validate.field).map.max_pairs = 10
  ];

  // Key constraints
  map<string, int32> scores = 2 [
    (buf.validate.field).map.keys.string = {min_len: 1, max_len: 64}
  ];

  // Value constraints
  map<string, int32> counts = 3 [
    (buf.validate.field).map.values.int32.gte = 0
  ];
}
```

## Oneof Rules

Use `(buf.validate.oneof).required = true` when the oneof represents a required choice (lookups, mutually exclusive options).
Omit `required` only when the oneof is intentionally optional.

```protobuf
message UserLookup {
  oneof value {
    option (buf.validate.oneof).required = true;
    string id = 1 [(buf.validate.field).string.uuid = true];
    string email = 2 [(buf.validate.field).string.email = true];
  }
}
```

## Timestamp Rules

```protobuf
import "google/protobuf/timestamp.proto";

message Event {
  google.protobuf.Timestamp created_at = 1 [(buf.validate.field).required = true];
  google.protobuf.Timestamp scheduled_at = 2 [(buf.validate.field).timestamp.gt_now = true];  // future
  google.protobuf.Timestamp occurred_at = 3 [(buf.validate.field).timestamp.lt_now = true];   // past
  google.protobuf.Timestamp expires_at = 4 [(buf.validate.field).timestamp.within = {seconds: 86400}];
}
```

## Duration Rules

```protobuf
import "google/protobuf/duration.proto";

message Config {
  google.protobuf.Duration timeout = 1 [
    (buf.validate.field).duration.gte = {seconds: 1},
    (buf.validate.field).duration.lte = {seconds: 300}
  ];
  google.protobuf.Duration interval = 2 [(buf.validate.field).duration.gt = {}];  // positive
}
```

## FieldMask Rules

```protobuf
import "google/protobuf/field_mask.proto";

message UpdateUserRequest {
  User user = 1;

  // Restrict allowed field paths
  google.protobuf.FieldMask update_mask = 2 [
    (buf.validate.field).field_mask = {in: ["name", "email", "bio"]}
  ];
}
```

Use `in` to specify the allowed field paths.

## Any Rules

```protobuf
import "google/protobuf/any.proto";

message Container {
  // Restrict to specific types
  google.protobuf.Any payload = 1 [
    (buf.validate.field).any = {
      in: ["type.googleapis.com/acme.User", "type.googleapis.com/acme.Group"]
    }
  ];
}
```

## Ignore Rules

```protobuf
message Request {
  // Skip all validation
  string internal = 1 [(buf.validate.field).ignore = IGNORE_ALWAYS];

  // Skip validation when field equals default value (useful for proto3 scalars)
  string optional_filter = 2 [(buf.validate.field).ignore = IGNORE_IF_ZERO_VALUE];
}
```

## Custom CEL Rules

CEL (Common Expression Language) enables complex validation logic. Within expressions, `this` refers to the field value (field rules) or entire message (message rules).

### Field-Level CEL

```protobuf
message Discount {
  int32 percentage = 1 [
    (buf.validate.field).cel = {
      id: "valid_percentage"
      message: "percentage must be between 0 and 100"
      expression: "this >= 0 && this <= 100"
    }
  ];

  // Multiple rules
  string promo_code = 2 [
    (buf.validate.field).cel = {
      id: "uppercase"
      message: "must be uppercase"
      expression: "this == this.upperAscii()"
    },
    (buf.validate.field).cel = {
      id: "alphanumeric"
      message: "must be alphanumeric"
      expression: "this.matches('^[A-Z0-9]+$')"
    }
  ];
}
```

### Message-Level CEL

```protobuf
message DateRange {
  option (buf.validate.message).cel = {
    id: "date_range_valid"
    message: "end_date must be after start_date"
    expression: "this.end_date > this.start_date"
  };

  google.protobuf.Timestamp start_date = 1;
  google.protobuf.Timestamp end_date = 2;
}

// Use has() to check field presence for conditional validation
message SearchFilters {
  option (buf.validate.message).cel = {
    id: "conditional_requirement"
    message: "max_price required when min_price is set"
    expression: "!has(this.min_price) || has(this.max_price)"
  };

  optional int32 min_price = 1;
  optional int32 max_price = 2;
}
```

## CEL Extension Functions

| Function | Description |
|----------|-------------|
| `isNan()`, `isInf()` | Test for NaN or infinity |
| `isEmail()`, `isHostname()`, `isUri()`, `isUriRef()` | Format validation |
| `isIp()`, `isIp(version)`, `isIpPrefix()` | IP/CIDR validation |
| `isHostAndPort()` | Host:port validation |
| `unique()` | Check list items are unique |
| `this` | Current value being validated |
| `now` | Current timestamp |

## Common Patterns

For entity and reference message patterns, see [best_practices.md](best_practices.md#common-patterns).

### Batch Request

```protobuf
message GetUsersRequest {
  repeated UserRef user_refs = 1 [
    (buf.validate.field).repeated.min_items = 1,
    (buf.validate.field).repeated.max_items = 250
  ];
}
```

### Pagination

```protobuf
message ListRequest {
  uint32 page_size = 1 [(buf.validate.field).uint32.lte = 250];
  string page_token = 2 [(buf.validate.field).string.max_len = 4096];
}
```

See `assets/proto/example/v1/book_service.proto` for a complete List example with response, ordering, and filtering.

### Cross-Reference Consistency

When the same identifier appears in multiple messages, all validation constraints must be identical.
See [best_practices.md](best_practices.md#cross-reference-consistency) for details and examples.

## Predefined Constraint Rules

When the same validation logic is repeated across many fields, define reusable predefined constraints.
Extension files must use proto2 or Editions syntax (not proto3).
Use field numbers 50000-99999 for private schemas.

```protobuf
// extensions.proto (proto2 or Editions)
extend buf.validate.StringRules {
  optional bool name = 50000 [(buf.validate.predefined).cel = {
    id: "string.name"
    message: "name must be 1-100 characters"
    expression: "this.size() >= 1 && this.size() <= 100"
  }];
}

// Usage in proto3 files
string first_name = 1 [(buf.validate.field).string.(acme.common.v1.name) = true];
```

For parameterized rules and more examples, see [protovalidate.com](https://protovalidate.com).

## Runtime Validation

Protovalidate annotations define constraints in the schema, but enforcement happens at runtime.
Add validation calls at your service boundaries—typically in RPC handlers or middleware.

**Go:**
```go
import protovalidate "github.com/bufbuild/protovalidate-go"

validator, err := protovalidate.New()
if err := validator.Validate(msg); err != nil {
    // Handle validation error
}
```

**TypeScript:**
```typescript
import { createValidator } from "@bufbuild/protovalidate";

const validator = createValidator();
const violations = validator.validate(msg);
```

For Python, Java, and other languages, see [protovalidate.com](https://protovalidate.com).
