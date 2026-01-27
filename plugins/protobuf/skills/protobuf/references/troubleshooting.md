# Troubleshooting

Common errors when working with Protocol Buffers, with solutions.

## Contents

- [Linting Errors](#linting-errors)
- [Breaking Change Errors](#breaking-change-errors)
- [Code Generation Errors](#code-generation-errors)
- [Import and Dependency Errors](#import-and-dependency-errors)
- [Field and Schema Evolution](#field-and-schema-evolution)
- [Runtime and Validation Errors](#runtime-and-validation-errors)

---

## Linting Errors

### Quick Reference

| Error | Problem | Fix |
|-------|---------|-----|
| ENUM_VALUE_PREFIX | `ACTIVE` | `STATUS_ACTIVE` (prefix with enum name) |
| ENUM_ZERO_VALUE_SUFFIX | `STATUS_UNKNOWN` | `STATUS_UNSPECIFIED` |
| PACKAGE_VERSION_SUFFIX | `acme.user` | `acme.user.v1` |
| FIELD_LOWER_SNAKE_CASE | `userName` | `user_name` |
| SERVICE_SUFFIX | `Users` | `UserService` |
| RPC_REQUEST_RESPONSE_UNIQUE | Shared request types | Unique `{Method}Request` per RPC |
| RPC_REQUEST_STANDARD_NAME | `UserRequest` | `GetUserRequest` |
| COMMENT_* | Missing comments | Add `//` comments on public elements |
| IMPORT_NO_WEAK/PUBLIC | `import public` | `import` (standard import) |

### Ignoring Lint Rules

For legitimate exceptions, use comments (unless `disallow_comment_ignores: true`):

```protobuf
// buf:lint:ignore ENUM_VALUE_PREFIX
enum LegacyStatus {
  UNKNOWN = 0;
  ACTIVE = 1;
}
```

Or configure in `buf.yaml`:

```yaml
lint:
  ignore_only:
    ENUM_VALUE_PREFIX:
      - proto/legacy
```

## Breaking Change Errors

Breaking changes are detected by `buf breaking --against <reference>`.
These errors indicate changes that would break existing clients.

### FIELD_NO_DELETE

```
// Before
message User {
  string id = 1;
  string email = 2;
  string name = 3;
}

// After - Error: field 3 deleted without reservation
message User {
  string id = 1;
  string email = 2;
}
```

**Fix:** Reserve deleted field numbers and names:

```protobuf
message User {
  reserved 3;
  reserved "name";

  string id = 1;
  string email = 2;
}
```

### FIELD_SAME_NUMBER

**Fix:** Never change field numbers. Create a new message version if restructuring is needed.

### FIELD_SAME_TYPE

**Fix:** Field types cannot change. Options:
1. Add a new field with the correct type, deprecate the old one
2. Create a new package version (v2)

```protobuf
message User {
  string user_id = 1 [deprecated = true];
  int64 user_id_v2 = 2;
}
```

### FIELD_SAME_JSON_NAME

**Fix:** Don't change JSON names. Use explicit `json_name` to preserve the original:

```protobuf
message User {
  string username = 1 [json_name = "userName"];  // Preserves JSON compatibility
}
```

### ENUM_VALUE_NO_DELETE

**Fix:** Reserve deleted enum values:

```protobuf
enum Status {
  reserved 2;
  reserved "STATUS_INACTIVE";

  STATUS_UNSPECIFIED = 0;
  STATUS_ACTIVE = 1;
}
```

### RPC_NO_DELETE

**Fix:** Don't remove RPCs. Deprecate instead:

```protobuf
// Deprecated: Use DeactivateUser instead.
rpc DeleteUser(DeleteUserRequest) returns (DeleteUserResponse) {
  option deprecated = true;
}
```

### MESSAGE_NO_DELETE / ENUM_NO_DELETE

**Fix:** Don't remove messages or enums that may be in use. Deprecate first:

```protobuf
option deprecated = true;
message LegacyUser { ... }
```

### FIELD_SAME_ONEOF

**Fix:** Never move existing fields into a oneof. Add new fields to the oneof instead:

```protobuf
message Request {
  string id = 1;  // Keep as-is
  oneof alternate_identifier {
    string name = 2;
    string email = 3;
  }
}
```

### Breaking Change Categories

Buf offers different strictness levels in `buf.yaml`:

```yaml
breaking:
  use:
    - FILE      # Strictest: file-level changes break
    - PACKAGE   # Package-level: allows moving between files
    - WIRE_JSON # Wire + JSON encoding only
    - WIRE      # Wire encoding only (most permissive)
```

For internal APIs, `WIRE_JSON` may be sufficient. For public APIs, use `FILE`.

## Code Generation Errors

### Buf: Plugin Not Found

**Fixes:**
1. Check plugin name spelling in `buf.gen.yaml`
2. For remote plugins, verify it exists on BSR: `buf.build/some/plugin`
3. For local plugins, ensure it's in PATH

### Buf: Remote Plugin Timeout

**Fix:** Retry the command. Transient network issues are common.

### Protoc: Plugin Not Found

**Fixes:**
1. Install the plugin
2. Ensure GOBIN is in PATH:
   ```bash
   export PATH="$PATH:$(go env GOPATH)/bin"
   ```

### Go: "go_package option required"

**Fixes:**

With buf (recommended): Enable managed mode in `buf.gen.yaml`:
```yaml
version: v2
managed:
  enabled: true
  override:
    - file_option: go_package_prefix
      value: github.com/yourorg/api/gen/go
```

With protoc: Add `go_package` to each proto file:
```protobuf
option go_package = "github.com/yourorg/api/gen/go/acme/user/v1;userv1";
```

### Go: Conflicting Package Names

**Fix:** Each proto package should map to a unique Go package.
Use buf managed mode to handle this automatically.

### Output Directory Issues

**Fixes:**
- Buf creates directories automatically—check write permissions
- Protoc requires directories to exist: `mkdir -p gen/go`

### Stale Generated Files

**Fix:** Clean and regenerate:
```bash
rm -rf gen/
buf generate
```

## Import and Dependency Errors

### Buf: Dependency Not Found

```
Error: import "buf/validate/validate.proto": not found
```

**Fix:** Add dependency to `buf.yaml` and update:

```yaml
deps:
  - buf.build/bufbuild/protovalidate
```

```bash
buf dep update
```

### Buf: Version Conflict

**Fix:** Run `buf dep update` to resolve.
If persists, check for conflicting version pins across dependencies.

### Protoc: Import Not Found

**Fixes:**

1. Add include path for well-known types:
   ```bash
   protoc -I /usr/local/include -I proto ...
   ```

2. Find where WKTs are installed:
   ```bash
   ls /usr/local/include/google/protobuf/
   ```

3. Vendor the dependency:
   ```bash
   git clone --depth 1 https://github.com/protocolbuffers/protobuf.git third_party/protobuf
   protoc -I third_party/protobuf/src -I proto ...
   ```

### Protoc: googleapis Import Not Found

**Fix:** Vendor googleapis or migrate to buf with BSR dependency:

```bash
git clone --depth 1 https://github.com/googleapis/googleapis.git third_party/googleapis
protoc -I third_party/googleapis -I proto ...
```

### Circular Import

**Fix:** Break the cycle by moving shared types to a common file:

```
# Before (circular)
a.proto imports b.proto
b.proto imports a.proto

# After (resolved)
common.proto (shared types)
a.proto imports common.proto
b.proto imports common.proto
```

## Field and Schema Evolution

### Reserved Field Conflicts

**Fix:** Use a different field number:

```protobuf
message User {
  reserved 3, 5;
  reserved "old_name";
  string name = 6;  // Not 3 or 5
}
```

### Field Number Reuse (Silent Corruption)

Reusing field numbers causes **silent data corruption**. Protobuf won't catch it:

```protobuf
// Version 1
message User {
  string email = 3;
}

// Version 2 - DANGEROUS: reused field number with different type
message User {
  int64 user_type = 3;  // Silently corrupts data!
}
```

**Prevention:**
1. Always reserve deleted field numbers
2. Run `buf breaking` in CI
3. Never reuse field numbers, even if types match

### Oneof Evolution

**Moving fields into oneof breaks wire compatibility:**

```protobuf
// Before
message Request {
  string id = 1;
}

// After - BREAKS clients
message Request {
  oneof ref {
    string id = 1;  // Same number, but now in oneof
    string name = 2;
  }
}
```

**Adding fields to existing oneof is safe.**

### Changing Field Optionality

Adding `optional` keyword changes generated code but is wire-compatible:

```protobuf
// Before - cannot distinguish unset from ""
string nickname = 2;

// After - can detect if set
optional string nickname = 2;
```

**With protovalidate:** Changing required validation is a behavioral breaking change.

### When to Create a New Version

Create a new package version (v1 → v2) when:
- Multiple breaking changes are needed
- The API shape is fundamentally changing
- Field type changes are required
- Deprecation period has ended

```protobuf
// acme/user/v1/user.proto - Keep for existing clients
package acme.user.v1;

// acme/user/v2/user.proto - New version with breaking changes
package acme.user.v2;
```

## Runtime and Validation Errors

### Protovalidate: Required Field Missing

**Fix:** Ensure required fields are populated before sending.

For optional fields that shouldn't fail validation when empty:

```protobuf
string url = 3 [
  (buf.validate.field).string.uri = true,
  (buf.validate.field).ignore = IGNORE_IF_DEFAULT_VALUE
];
```

### Protovalidate: Pattern Mismatch

**Fix:** Ensure input matches the regex. Check for:
- Uppercase letters when lowercase required
- Invalid characters
- Missing required prefix/suffix

### Unknown Fields Warning

**Causes:**
- Client using newer proto than server
- Proto files out of sync between services

**Fix:** Regenerate and redeploy to ensure consistent proto versions.

### Message Size Limits

**Fixes:**
1. Increase limit if appropriate:
   ```go
   grpc.MaxRecvMsgSize(16 * 1024 * 1024)
   ```
2. Use streaming for large payloads
3. Paginate responses
