# Validation and CEL (reference)

CRD schema validation is part of the API contract. The goal is to prevent invalid stored objects, not just to improve documentation.

## Baseline: required + enums

- Mark truly required invariants as required.
- Use enums for constrained strings.

### Example

```yaml
properties:
  spec:
    type: object
    required: [mode]
    properties:
      mode:
        type: string
        enum: ["Auto", "Manual"]
      replicas:
        type: integer
        minimum: 0
```

## Cross-field constraints: CEL (`x-kubernetes-validations`)

Use CEL when invalid combinations cannot be expressed with simple OpenAPI rules.

### Example: `replicas` only allowed in Manual mode

```yaml
properties:
  spec:
    type: object
    properties:
      mode:
        type: string
        enum: ["Auto", "Manual"]
      replicas:
        type: integer
        minimum: 0
    x-kubernetes-validations:
      - rule: "self.mode == 'Manual' || !has(self.replicas)"
        message: "spec.replicas is only allowed when spec.mode is Manual"
```

### Example: exactly one of two fields

```yaml
x-kubernetes-validations:
  - rule: "has(self.inline) != has(self.secretRef)"
    message: "exactly one of spec.inline or spec.secretRef must be set"
```

## Review checklist

- Are required fields truly required for all valid objects?
- Are defaults compatible with existing clients and semantics?
- Are nullable fields used intentionally?
- Are CEL rules minimal, targeted, and accompanied by clear messages?

## Compatibility warnings

- Tightening validation can be a breaking change for existing stored objects.
- Defaulting changes can be a semantic breaking change even if schema types stay constant.

