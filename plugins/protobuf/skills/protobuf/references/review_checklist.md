# Proto API Review Checklist

Use this checklist when reviewing `.proto` files before an API stabilizes.
It captures common issues found during real-world proto API reviews.

Every field in a production API should have [protovalidate](protovalidate.md) constraints.
A field without validation is a field that accepts anything.

## Contents

- [Enum Validation](#enum-validation)
- [Oneof Validation](#oneof-validation)
- [String Pattern Validation](#string-pattern-validation)
- [Cross-Reference Consistency](#cross-reference-consistency)
- [Pagination Validation](#pagination-validation)
- [Header and Metadata Fields](#header-and-metadata-fields)
- [List Request Completeness](#list-request-completeness)
- [Reserved Statements](#reserved-statements)
- [Documentation Quality](#documentation-quality)

---

## Enum Validation

Every enum field should have appropriate validation constraints.
The correct constraints depend on how the enum is used.

Proto3 enums should have a zero value with the `_UNSPECIFIED` suffix (e.g., `STATUS_UNSPECIFIED = 0`).
This value represents "not set" and should have no semantic meaning.
See [Enums](best_practices.md#enums) for naming conventions.

| Context | Constraints | Rationale |
|---------|-------------|-----------|
| Required enum field | `not_in = 0` + `defined_only = true` | Rejects the zero value and unknown values |
| Optional enum field (zero value = "not set") | `defined_only = true` only | Zero value means "not set" or "no preference" |
| Repeated enum items | `.repeated.items.enum.not_in = 0` + `.items.enum.defined_only = true` | Each item must be meaningful |

**Common mistake:** Using only `defined_only = true` on a required enum field.
This allows the zero value through, which is rarely the intent.

```protobuf
// WRONG: allows the zero value through
Status status = 1 [(buf.validate.field).enum.defined_only = true];

// RIGHT: rejects both the zero value and unknown values
Status status = 1 [
  (buf.validate.field).enum.not_in = 0,
  (buf.validate.field).enum.defined_only = true
];

// RIGHT: optional enum where zero means "no preference"
Status status_filter = 2 [(buf.validate.field).enum.defined_only = true];
```

## Oneof Validation

All oneofs representing a required choice should have `(buf.validate.oneof).required = true`.

**Check these oneof types:**

- **Lookup messages** (id-or-name lookups): always required
- **Mutually exclusive operations** (add/update/delete in diff messages): always required
- **Value types** (string/int/bool variants): required unless the oneof is intentionally optional

```protobuf
// Lookup by ID or email - always required
message UserLookup {
  oneof value {
    option (buf.validate.oneof).required = true;
    string id = 1 [(buf.validate.field).string.uuid = true];
    string email = 2 [(buf.validate.field).string.email = true];
  }
}
```

**Common mistake:** Omitting `required = true` on a lookup oneof.
This allows clients to send a lookup with no value set, which is never valid.

## String Pattern Validation

Name and identifier fields should have pattern constraints.
Choose the right pattern for the field type.

| Field Type | Pattern | Example Values |
|-----------|---------|----------------|
| Lowercase name with hyphens | `^[a-z0-9][a-z0-9-]*[a-z0-9]$` | `my-source`, `order-api` |
| Versioned label (dots, underscores, hyphens) | `^[a-z0-9]([a-z0-9._-]*[a-z0-9])?$` | `v1.0.0-rc1`, `main` |
| Programming identifier | `^[_a-zA-Z][_a-zA-Z0-9]*$` | `my_variable`, `userId` |
| PascalCase identifier | `^[A-Za-z][A-Za-z0-9]*$` | `GetUser`, `ListOrders` |

**Review items:**

- [ ] Name fields have a `pattern` constraint appropriate for the entity type
- [ ] Consider whether `min_len` and `max_len` constraints are appropriate
- [ ] Patterns match the documented naming rules for the entity type
- [ ] Patterns are consistent with the character set the spec allows

## Cross-Reference Consistency

When the same identifier appears in multiple messages, all validation constraints must match.

**Example problem:** `User.name` has `max_len = 32` but `TeamMember.user_name` (which references the same user name) has `max_len = 64`.
A team member could reference a user name that is too long to be a valid user name.

**Review items:**

- [ ] Find all messages that reference the same identifier
- [ ] Verify `max_len`, `min_len`, and `pattern` are identical across all references
- [ ] Check companion messages for consistency

## Pagination Validation

Pagination fields should have validation constraints.

```protobuf
uint32 page_size = 1 [(buf.validate.field).uint32.lte = 250];
string page_token = 2 [(buf.validate.field).string.max_len = 4096];
```

**Review items:**

- [ ] `page_size` has an upper bound
- [ ] `page_token` has a `max_len`
- [ ] Response includes `next_page_token` with the same `max_len`
- [ ] Response repeated field has `max_items` matching the `page_size` upper bound

## Header and Metadata Fields

Key-value string fields without bounds are a common source of abuse.

```protobuf
message Header {
  string key = 1 [
    (buf.validate.field).required = true,
    (buf.validate.field).string.max_len = 256
  ];
  string value = 2 [(buf.validate.field).string.max_len = 8192];
}
```

**Review items:**

- [ ] Header/metadata key and value fields have `max_len`
- [ ] Unbounded `string` fields are intentionally unbounded, not accidentally so

## List Request Completeness

List RPCs should follow a consistent pattern across the API.

**Review items:**

- [ ] All List requests have pagination (page_size + page_token)
- [ ] All List requests have an Order enum
- [ ] Order enum includes both `CREATE_TIME_DESC/ASC` and `UPDATE_TIME_DESC/ASC` when the entity has both timestamps
- [ ] Order enum values follow the `ORDER_*` naming pattern
- [ ] Default order is documented
- [ ] Useful entity-specific optional fields are present (e.g., by parent, by status)
- [ ] Optional enum fields use `defined_only = true` only (zero value = "no preference")

```protobuf
message ListEntitiesRequest {
  enum Order {
    ORDER_UNSPECIFIED = 0;
    ORDER_CREATE_TIME_DESC = 1;
    ORDER_CREATE_TIME_ASC = 2;
    ORDER_UPDATE_TIME_DESC = 3;
    ORDER_UPDATE_TIME_ASC = 4;
  }

  uint32 page_size = 1 [(buf.validate.field).uint32.lte = 250];
  string page_token = 2 [(buf.validate.field).string.max_len = 4096];
  Order order = 3 [(buf.validate.field).enum.defined_only = true];
}
```

## Reserved Statements

When field numbers are skipped, always add `reserved` statements to prevent accidental reuse.

**Review items:**

- [ ] No gaps in field numbering without a `reserved` statement
- [ ] Removed fields have both the field number and name reserved
- [ ] Oneofs that start at field 2+ have field 1 reserved (or documented why)

```protobuf
message Example {
  reserved 1;

  // Type of the value.
  ValueType type = 2;

  oneof value {
    string string_value = 3;
    int64 int_value = 4;
  }
}
```

## Documentation Quality

Documentation errors erode trust in the API and confuse implementers.

**Review items:**

- [ ] **Grammar: articles before vowels** — "an Environment" not "a Environment", "an Image" not "a Image"
- [ ] **Consistent terminology** — pick one form and use it everywhere (e.g., "shortname" not sometimes "short name" and sometimes "shortname")
- [ ] **No stale references** — check for references to old entity/concept names that have been renamed
- [ ] **Implicit behavior is documented** — if an entity is auto-created, has reserved names, or has special behavior, document it on the relevant field
- [ ] **Complete sentences** — comments should be full sentences with periods
- [ ] **No truncated comments** — watch for comments cut off mid-sentence (e.g., "typically by the" with no completion)
- [ ] **Typos** — common in large proto reviews: run spellcheck or read comments carefully
