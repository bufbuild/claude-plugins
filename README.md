![The Buf logo](https://raw.githubusercontent.com/bufbuild/protovalidate/main/.github/buf-logo.svg)

# Buf Plugins for Claude Code

[![License](https://img.shields.io/github/license/bufbuild/claude-plugins?color=blue&v=1)][license]
[![Slack](https://img.shields.io/badge/Slack-Buf-%23e01563)][slack]

Official [Claude Code][claude-code] plugins from [Buf][buf] for Protocol Buffers, Connect, and BSR development.

## Installation

Add this marketplace to Claude Code:

```sh
/plugin marketplace add https://github.com/bufbuild/claude-plugins
```

Install a plugin:

```sh
/plugin install protobuf@buf-plugins
```

## Available Plugins

### protobuf

Protocol Buffers development with Buf toolchain and best practices.

- **Schema design**: Field types, enums, oneofs, maps, evolution patterns
- **Code generation**: Templates for Go (gRPC/Connect), TypeScript, Python, Java
- **Validation**: [Protovalidate][protovalidate] constraint patterns
- **Tooling**: buf CLI, troubleshooting, migration from protoc

Triggers on: `*.proto`, `buf.yaml`, `buf.*.yaml`, `buf.gen.yaml`, `buf.gen.*.yaml`, `buf.lock`

## Documentation

- [Buf Documentation][buf-docs]
- [Protovalidate][protovalidate]
- [Connect RPC][connectrpc]

## Evals and production telemetry

The `evals/protobuf/` directory contains a small human-review eval set for
schema evolution, Protovalidate design, and Buf configuration review. The cases
are harness-neutral so plugin behavior can be checked before release in Claude
Code or another agent workspace.

If you publish this plugin through Telvine, keep runtime telemetry metadata-only:
`skill.invocation.start`, `skill.invocation.end`, and `skill.invocation.error`
for skill behavior, plus `plugin.component.invoked` and
`plugin.component.error` for non-skill components. Do not emit private schemas,
repository contents, generated code, connector payloads, tool arguments, or
model outputs.

## Community

For help and discussion around Protobuf, best practices, and more, join us on [Slack][slack].

## Legal

Offered under the [Apache 2 license][license].

[buf]: https://buf.build
[buf-docs]: https://buf.build/docs
[claude-code]: https://claude.ai/code
[connectrpc]: https://connectrpc.com
[license]: LICENSE
[protovalidate]: https://protovalidate.com
[slack]: https://buf.build/links/slack
