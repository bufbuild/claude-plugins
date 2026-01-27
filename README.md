![The Buf logo](https://raw.githubusercontent.com/bufbuild/protovalidate/main/.github/buf-logo.svg)

# Buf Plugins for Claude Code

[![License](https://img.shields.io/github/license/bufbuild/claude-plugins?color=blue)][license]
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
