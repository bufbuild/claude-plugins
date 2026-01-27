# Migrating from Protoc to Buf

Guide for converting protoc-based projects to buf.
Based on the [official migration guide](https://buf.build/docs/migration-guides/migrate-from-protoc/).

## Contents

- [Key Concept: Include Paths to Modules](#key-concept-include-paths-to-modules)
- [Step 1: Create buf.yaml](#step-1-create-bufyaml)
- [Step 2: Verify Compilation](#step-2-verify-compilation)
- [Step 3: Create buf.gen.yaml](#step-3-create-bufgenyaml)
- [Step 4: Generate Code](#step-4-generate-code)
- [Step 5: Replace Vendored Dependencies with BSR](#step-5-replace-vendored-dependencies-with-bsr)
- [Step 6: Enable Managed Mode](#step-6-enable-managed-mode)
- [Step 7: Use Remote Plugins](#step-7-use-remote-plugins)
- [Step 8: Update Build Scripts](#step-8-update-build-scripts)
- [Migration Checklist](#migration-checklist)
- [Troubleshooting](#troubleshooting)

---

## Key Concept: Include Paths to Modules

The fundamental shift: protoc's `-I` include paths become Buf's module paths. With buf, there is no `-I` flag—each protoc `-I` path maps to a `path` field in buf.yaml.

## Step 1: Create buf.yaml

Place a `buf.yaml` at your workspace root. Convert each `-I` path to a module path:

**Before (protoc):**
```bash
protoc \
  -I proto \
  -I vendor/googleapis \
  -I vendor/protoc-gen-validate \
  --go_out=gen ...
```

**After (buf.yaml):**
```yaml
version: v2
modules:
  - path: proto
  - path: vendor/googleapis
  - path: vendor/protoc-gen-validate
lint:
  use:
    - STANDARD
breaking:
  use:
    - FILE
```

## Step 2: Verify Compilation

Test that your workspace compiles:

```bash
buf build
```

This discovers `.proto` files, compiles them in memory, and validates the build succeeds.

## Step 3: Create buf.gen.yaml

Replace protoc plugin invocations with buf.gen.yaml:

**Before (protoc):**
```bash
protoc \
  -I proto \
  -I vendor/googleapis \
  --go_out=gen \
  --go_opt=paths=source_relative \
  --go-grpc_out=gen \
  --go-grpc_opt=paths=source_relative \
  proto/**/*.proto
```

**After (buf.gen.yaml):**
```yaml
version: v2
plugins:
  - local: protoc-gen-go
    out: gen
    opt:
      - paths=source_relative
  - local: protoc-gen-go-grpc
    out: gen
    opt:
      - paths=source_relative
```

## Step 4: Generate Code

```bash
buf generate
```

## Step 5: Replace Vendored Dependencies with BSR

Once basic migration works, replace vendored third-party protos with BSR dependencies:

```yaml
# buf.yaml
version: v2
modules:
  - path: proto
deps:
  - buf.build/googleapis/googleapis
  - buf.build/bufbuild/protovalidate
```

Then run:
```bash
buf dep update
```

Delete the vendored directories after confirming generation still works.

## Step 6: Enable Managed Mode

Managed mode is the recommended approach—it lets you remove language-specific options from proto files:

```yaml
# buf.gen.yaml
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
```

With managed mode enabled, you can remove `go_package` options from your proto files.

## Step 7: Use Remote Plugins

Remote plugins are the recommended default—no local installation needed:

```yaml
plugins:
  # Before: local plugin
  - local: protoc-gen-go
    out: gen

  # After: remote plugin (no installation needed)
  - remote: buf.build/protocolbuffers/go
    out: gen
```

## Step 8: Update Build Scripts

**Before:**
```makefile
generate:
	./scripts/generate.sh
```

**After:**
```makefile
.PHONY: all lint format generate breaking

all: format lint generate

lint:
	buf lint

format:
	buf format -w

generate:
	buf generate

breaking:
	buf breaking --against '.git#branch=main'
```

## Multiple Generation Templates

For projects with different API subsets:

```bash
buf generate proto/public --template buf.public.gen.yaml
buf generate proto/internal --template buf.internal.gen.yaml
```

## Migration Checklist

- [ ] Create `buf.yaml` with modules matching `-I` paths
- [ ] Run `buf build` to verify compilation
- [ ] Create `buf.gen.yaml` matching protoc plugin options
- [ ] Run `buf generate` and compare output
- [ ] Replace vendored deps with BSR deps (`buf dep update`)
- [ ] Enable managed mode and remove `go_package` options
- [ ] Switch to remote plugins
- [ ] Update Makefile/scripts
- [ ] Add `buf lint` and `buf breaking` to CI
- [ ] Delete vendored protos and old scripts
- [ ] Commit `buf.lock`

## Troubleshooting

### Import Not Found After Migration

Ensure all `-I` paths are represented as modules in buf.yaml. BSR dependencies use canonical paths:

```protobuf
// Vendored import
import "vendor/googleapis/google/api/annotations.proto";

// BSR import (after migration)
import "google/api/annotations.proto";
```

### Different Generated Output

Remote plugins may produce slightly different output than local versions. During migration, use local plugins for exact compatibility, then switch to remote plugins.

### go_package Still Required

If some files must keep `go_package` (e.g., files you don't control), disable managed mode for those paths:

```yaml
managed:
  enabled: true
  disable:
    - file_option: go_package
      path: vendor
```
