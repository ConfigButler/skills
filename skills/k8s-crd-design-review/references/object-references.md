# Object references & relationships (reference)

This reference summarizes Kubernetes API conventions for fields that point at other objects.

## Source

Derived from the upstream Kubernetes API conventions (see “Object references” section):
https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md

## Key conventions (review checklist)

### Naming

- If a field refers to another object via a structured reference, use `fooRef` (one-to-one relationship)
- For lists of references, use `fooRefs` (one-to-many relationship)

The suffix `Ref`/`Refs` is intentionally consistent; avoid ad-hoc names like `fooReference`, `fooObject`, `fooTarget`, etc., unless the purpose demands it (e.g. `targetRef`).

### When is name-only (`fooName`) the right choice?

Prefer a structured reference (`fooRef`) by default when you can: being explicit in the schema tends to age better, is friendlier to tooling (e.g., editor plugins that allow to “jump to definition”), and keeps future evolution additive.

Using a plain string name is still **allowed**, but treat it as an optimization for very constrained cases where **the controller can unambiguously infer everything else**. 

`fooName` is ok-ish when the following are all true:

- The reference is **always to exactly one resource type** (controller has a fixed GVR / kind).
- The reference is **same-namespace** (or the referenced object is cluster-scoped), so `namespace` does not need to be expressed.
- You do not need qualifiers like `group`, `resource`, `kind`, or `fieldPath`.
- You do not expect the reference to evolve into multi-kind or cross-namespace.

If there is any reasonable chance you will later want qualifiers like `group`/`resource`/`kind`, cross-namespace support, multi-kind support, or better UX for “reference browsing”, start with `fooRef`. Changing a field from `fooName: string` to `fooRef: object` is typically a **breaking API change**.

### Namespace scope

- For **namespaced** resources, references should normally be **same-namespace**.
- Cross-namespace references are discouraged because namespaces are security boundaries.
  - If you allow them anyway, explicitly document semantics (creation ordering, deletion behavior, permissions), and consider admission-time permission checks or double opt-in.

### Schema shape

- Prefer a structured object over a bare string when the reference may need to evolve (e.g., from single-kind to multi-kind).
- Avoid including `apiVersion` in references unless the reference is specifically to a field in a particular versioned schema.

### Defaults & requiredness (practical guidance)

- Do not default the *identity* (`name`). A reference without a `name` is not actionable.
- Defaulting *qualifiers* like `group` / `resource` / `kind` **can be a good UX and tooling-friendly design** when the reference is currently single-kind and you want users to be able to write:

  ```yaml
  targetRef:
    name: my-name
  ```

  This keeps the schema explicit enough that tools can infer what is being referenced, while keeping the authoring surface small.

- Treat defaults as part of the API contract:
  - Changing defaults later can be a semantic breaking change.
  - If you anticipate expanding to additional target types in the future, you can keep the existing defaults for backward compatibility and add optional fields/validation to support additional types.

- If a reference is required for the resource to be meaningful, make the reference field required (e.g., require `fooRef` or `fooName` in `spec`).
- Within a `*Ref` object, require the identifying parts (always `name`; and for multi-kind refs also `group` + `resource`, or `kind` only if your implementation has a safe, non-ambiguous mapping).
- Prefer omitting `namespace` (assume same-namespace) over defaulting it. If you do include `namespace`, treat it as an explicit, security-sensitive choice.

Example (single-kind today, future-proof for later):

```yaml
targetRef:
  properties:
    group:
      type: string
      default: example.com
    kind:
      type: string
      default: GitTarget
      enum: [GitTarget]
    name:
      type: string
  required: [name]
```

## Recommended schemas

### Single-kind reference (controller knows GVR)

Use when the reference is **always** to a single resource type.

```yaml
spec:
  secretRef:
    name: my-secret
    # namespace is usually omitted; cross-namespace is discouraged
```

### Multi-kind reference (bounded set of supported types)

Use when the reference can point to more than one type.

```yaml
spec:
  targetRef:
    group: example.com
    resource: widgets
    name: my-widget
    # namespace is typically omitted or assumed same-namespace
```

Notes:

- Prefer `resource` over `kind` when the mapping could be ambiguous across API groups/resources.
- Including `group` avoids ambiguity and helps copy/paste portability.

## Controller behavior guidance

- Assume the referenced object might not exist; surface a clear error via Conditions/Events.
- Validate reference fields before using them as API path segments.
- Do **not** modify the referenced object (avoid privilege escalation vectors).
- Minimize copying values from the referenced object into the referrer (including `status` and Events/logs), to avoid leaking information a user may not have permission to read.

## Common review flags

- Reference fields not suffixed with `Ref`/`Refs`.
- Cross-namespace references without explicit semantics and guardrails.
- Free-form strings used for “references” where a structured schema is expected.
- Status/spec fields that echo data read from the referenced object without a clear, safe rationale.
