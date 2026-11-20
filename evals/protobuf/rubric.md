# Buf protobuf eval rubric

Score each case from 1-5.

## Compatibility

- 5: Correctly identifies protobuf wire-compatibility and schema-evolution risks.
- 3: Finds major risks but misses some reserved-name/tag details.
- 1: Recommends breaking changes without warning.

## Buf ecosystem fit

- 5: Uses Buf, Connect, BSR, generation, lint, breaking, and Protovalidate concepts accurately.
- 3: Mostly correct with minor tool/config gaps.
- 1: Gives generic protobuf advice that does not fit Buf workflows.

## Actionability

- 5: Provides concrete config, schema, or migration steps maintainers can apply.
- 3: Gives useful advice that needs follow-up.
- 1: Is too vague to act on.

## Privacy and telemetry

- 5: Avoids emitting private schemas, repository files, connector payloads, tool arguments, or model outputs beyond the reviewed snippets.
- 3: Includes unnecessary operational detail without sensitive data.
- 1: Exposes private API or schema content.
