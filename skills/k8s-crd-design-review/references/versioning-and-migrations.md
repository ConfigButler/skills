# Versioning and migrations (reference)

Every CRD change must be assessed for compatibility impact and whether users need a migration plan.

## Breaking change taxonomy (common)

Schema and contract breaks:

- Removing a field
- Renaming a field
- Type changes (string → int, object → string, etc.)
- Tightening validation (e.g., new required fields, narrower enums, new CEL rules)
- Changing defaulting behavior
- Changing list semantics (atomic → map, key changes, order meaning)

Behavioral breaks (even if schema looks similar):

- Controller interprets existing values differently
- Reconciliation changes that invalidate old objects
- Status semantics changes that break automation

## Deprecation playbook (field-level)

Use a multi-release approach:

1. Introduce new field(s) alongside old
2. Keep old field(s) working
3. Emit warnings / Conditions / Events indicating deprecation
4. Update docs/examples and clients
5. Eventually remove old field(s) in a major version (or when policy allows)

## Versioning strategies

- Serve multiple versions when you need to evolve the schema without forcing immediate migrations.
- Use conversion webhooks when versions are meaningfully different.

## Storage version reminders

- Storage version changes require a plan for existing stored objects.
- Ensure conversion is correct in both directions for all served versions.

## Review checklist

- Is this change safe for existing stored objects?
- If not, is there an explicit migration plan?
- Are rollout steps and communication/deprecation steps defined?

