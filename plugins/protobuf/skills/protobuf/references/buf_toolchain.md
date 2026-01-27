# Buf Toolchain Reference

Reference for the buf CLI and configuration files.
For proto design patterns, see [best_practices.md](best_practices.md).

**Documentation:** [buf.build/docs](https://buf.build/docs)

## Contents

- [CLI Commands](#cli-commands)
- [buf.yaml Configuration](#bufyaml-configuration)
- [buf.gen.yaml Configuration](#bufgenyaml-configuration)
- [buf.lock](#buflock)
- [Project Structure](#project-structure)
- [BSR (Buf Schema Registry)](#bsr-buf-schema-registry)
- [Build Integration](#build-integration)

---

## CLI Commands

### Quick Reference

| Task | Command |
|------|---------|
| Lint | `buf lint` |
| Format (check) | `buf format --diff` |
| Format (apply) | `buf format -w` |
| Breaking check | `buf breaking --against '.git#branch=main'` |
| Generate | `buf generate` |
| Build | `buf build` |
| Push to BSR | `buf push` |
| Update deps | `buf dep update` |
| Prune deps | `buf dep prune` |
| Login | `buf registry login` |

**Check project Makefile/package.json first**—projects often wrap these commands with project-specific options.

### Breaking Change Detection

```bash
# Compare against git main branch
buf breaking --against '.git#branch=main'

# Compare against remote module
buf breaking --against 'buf.build/acme/api'

# Compare against specific git commit
buf breaking --against '.git#ref=abc123'

# Compare local directories
buf breaking --against ../previous-version
```

## buf.yaml Configuration

The `buf.yaml` file configures modules, linting, and breaking change detection.

### Basic Structure

```yaml
version: v2
modules:
  - path: proto
    name: buf.build/yourorg/yourmodule
lint:
  use:
    - STANDARD
breaking:
  use:
    - FILE
```

### Lint Configuration

The STANDARD rule set includes PROTOVALIDATE rules—no need to add both:

```yaml
# Correct
lint:
  use:
    - STANDARD

# Redundant
lint:
  use:
    - STANDARD
    - PROTOVALIDATE  # Already included in STANDARD
```

Additional options:

```yaml
lint:
  use:
    - STANDARD           # Recommended baseline
    - COMMENTS           # Require comments on all public elements
    - UNARY_RPC          # Disallow streaming RPCs

  except:
    - PACKAGE_VERSION_SUFFIX  # Allow packages without version

  ignore:
    - proto/internal          # Ignore paths

  ignore_only:
    ENUM_VALUE_PREFIX:
      - proto/legacy          # Ignore specific rule in specific paths

  # Prevent buf:lint:ignore comments
  disallow_comment_ignores: true
```

### Breaking Change Configuration

```yaml
breaking:
  use:
    - FILE        # Strictest: detects file-level breaks
    # - PACKAGE   # Package-level (allows moving between files)
    # - WIRE_JSON # Wire + JSON encoding breaks only
    # - WIRE      # Wire encoding breaks only

  except:
    - FIELD_SAME_JSON_NAME  # Allow JSON name changes

  ignore:
    - proto/internal        # Ignore paths
```

### Multi-Module Workspace

```yaml
version: v2
modules:
  - path: proto/public
    name: buf.build/yourorg/public-api
  - path: proto/internal
    name: buf.build/yourorg/internal-api
```

### Dependencies

```yaml
version: v2
modules:
  - path: proto
deps:
  - buf.build/googleapis/googleapis
  - buf.build/bufbuild/protovalidate
```

After adding dependencies, run `buf dep update` to generate/update `buf.lock`.

## buf.gen.yaml Configuration

The `buf.gen.yaml` file configures code generation.

### Basic Structure with Remote Plugins

Remote plugins are the recommended default—no local installation needed:

```yaml
version: v2
plugins:
  - remote: buf.build/protocolbuffers/go
    out: gen
    opt:
      - paths=source_relative
```

### Managed Mode

Managed mode centralizes package naming configuration, keeping `.proto` files language-agnostic.
This eliminates the need for `go_package`, `java_package`, etc. in proto files.

```yaml
version: v2
managed:
  enabled: true
  override:
    - file_option: go_package_prefix
      value: github.com/yourorg/yourrepo/gen
```

**Common managed mode options:**

| Option | Description |
|--------|-------------|
| `go_package_prefix` | Prefix for Go import paths |
| `java_package_prefix` | Prefix for Java packages (prepended to proto package) |
| `java_multiple_files` | Generate separate file per message (recommended: `true`) |
| `csharp_namespace_prefix` | Prefix for C# namespaces |
| `ruby_package_suffix` | Suffix for Ruby packages |
| `objc_class_prefix` | Prefix for Objective-C classes |

### Language-Specific Configurations

#### Go with Connect (Recommended)

```yaml
version: v2
managed:
  enabled: true
  override:
    - file_option: go_package_prefix
      value: github.com/yourorg/api/gen
plugins:
  - remote: buf.build/protocolbuffers/go
    out: gen
    opt:
      - paths=source_relative
  - remote: buf.build/connectrpc/go
    out: gen
    opt:
      - paths=source_relative
```

**Variants:**
- **gRPC:** Replace `buf.build/connectrpc/go` with `buf.build/grpc/go`
- **Opaque API (Go 1.21+):** Add `default_api_level=API_OPAQUE` to protocolbuffers/go options
- **With protovalidate:** Add `disable` block for `buf.build/bufbuild/protovalidate` to prevent managed mode from overwriting its go_package

#### TypeScript with Connect

```yaml
version: v2
plugins:
  - remote: buf.build/bufbuild/es
    out: gen
    opt:
      - target=ts
  - remote: buf.build/connectrpc/es
    out: gen
    opt:
      - target=ts
```

Options: `target=ts` (TypeScript), `target=js` (CommonJS), `target=js+dts` (JS + declarations)

#### Python with gRPC

```yaml
version: v2
plugins:
  - remote: buf.build/protocolbuffers/python
    out: gen
  - remote: buf.build/grpc/python
    out: gen
```

Add `buf.build/protocolbuffers/pyi` for type stubs.

#### Java with gRPC

```yaml
version: v2
managed:
  enabled: true
  override:
    - file_option: java_package_prefix
      value: com.yourorg.api
    - file_option: java_multiple_files
      value: true
plugins:
  - remote: buf.build/protocolbuffers/java
    out: gen
  - remote: buf.build/grpc/java
    out: gen
```

**Kotlin:** Use `buf.build/grpc/kotlin` instead of `buf.build/grpc/java`.

#### Other Languages

For C#, Swift, Rust, and other languages, see [buf.build/docs/generate/overview](https://buf.build/docs/generate/overview).

### Local Plugins

Use local plugins when remote plugins aren't available or for custom generators:

```yaml
plugins:
  - local: protoc-gen-custom
    out: gen
```

### Input Filtering

Control which protos are generated:

```yaml
version: v2
inputs:
  - directory: proto
    paths:
      - acme/user/v1      # Include only this package
    exclude_paths:
      - acme/internal     # Exclude this package

plugins:
  - remote: buf.build/protocolbuffers/go
    out: gen
```

### Clean Output Directory

Delete generated files before regenerating:

```yaml
version: v2
clean: true
plugins:
  - remote: buf.build/protocolbuffers/go
    out: gen
```

The `clean: true` option removes stale generated files before regenerating. This prevents
orphaned files when protos are renamed or deleted.

**Caution:** Only use `clean: true` if the output directory contains exclusively generated
code—it will delete all existing files in the output directory before regeneration.

## buf.lock

Auto-generated by `buf dep update`. Locks dependency versions for reproducible builds.

- Commit `buf.lock` to version control
- Run `buf dep update` when changing deps in `buf.yaml`
- Run `buf dep prune` to remove unused deps

## Project Structure

### Single Module

```
project/
├── buf.yaml
├── buf.gen.yaml
├── buf.lock
└── proto/
    └── acme/
        └── user/
            └── v1/
                ├── user.proto
                └── user_service.proto
```

### Multi-Module Workspace

```
project/
├── buf.yaml           # Workspace config with multiple modules
├── buf.gen.yaml
├── buf.lock
├── proto/
│   ├── public/        # Public API module
│   │   └── acme/
│   │       └── api/
│   │           └── v1/
│   └── internal/      # Internal module
│       └── acme/
│           └── internal/
│               └── v1/
```

## BSR (Buf Schema Registry)

### Module Naming

BSR modules follow the pattern: `buf.build/<owner>/<repository>`

- `buf.build/googleapis/googleapis` - Google APIs
- `buf.build/bufbuild/protovalidate` - Protovalidate
- `buf.build/yourorg/yourapi` - Your organization's API

### Consuming BSR Modules

Add to `buf.yaml` dependencies:

```yaml
deps:
  - buf.build/googleapis/googleapis
  - buf.build/bufbuild/protovalidate
```

Import in proto files:

```protobuf
import "google/api/annotations.proto";
import "buf/validate/validate.proto";
```

### Publishing to BSR

1. Configure module name in `buf.yaml`:
   ```yaml
   modules:
     - path: proto
       name: buf.build/yourorg/yourapi
   ```

2. Authenticate: `buf registry login`

3. Push: `buf push`

## Build Integration

**Check for existing commands first.** Many projects use Makefile, package.json scripts, or other build tools to wrap buf commands with project-specific options.

Common patterns:
- `make lint` / `make format` / `make generate`
- `npm run proto:lint` / `npm run proto:generate`
- Task runners, Bazel, etc.

Example Makefile if none exists: [Makefile.example](../assets/Makefile.example)
