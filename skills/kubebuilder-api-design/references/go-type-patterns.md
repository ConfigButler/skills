# Go type patterns for Kubernetes APIs

Use this file when mapping an API shape into idiomatic Kubernetes Go types in Kubebuilder projects.

## Contents

- Optional vs required fields
- Time, duration, quantity
- Object references (namespaced/name)
- Enums and unions (best-effort)

## Optional vs required (Go struct design)

General rule: represent “optional” as `omitempty` plus a pointer for scalars.

- Required scalar: `int32`, `string`, `bool` (no `omitempty`)
- Optional scalar: `*int32`, `*string`, `*bool` with `omitempty`

Why: preserves distinction between “unset” and “set to zero/false/empty”.

For optional structs, prefer pointers as well:

```go
type FooSpec struct {
  Config *FooConfig `json:"config,omitempty"`
}
```

## Time and durations

- Timestamps: `metav1.Time`
- Durations: `metav1.Duration` (serializes as string like `"5s"`)

Canonical Kubebuilder background: [`book.kubebuilder.io/cronjob-tutorial/api-design.html`](https://book.kubebuilder.io/cronjob-tutorial/api-design.html).

```go
import metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"

type FooStatus struct {
  LastTransitionTime *metav1.Time `json:"lastTransitionTime,omitempty"`
}
```

## Resource quantities

Use `resource.Quantity` for CPU/memory/storage-like values:

```go
import "k8s.io/apimachinery/pkg/api/resource"

type FooSpec struct {
  // e.g. "100m", "1", "1Gi"
  CPU resource.Quantity `json:"cpu"`
}
```

Notes:

- Avoid `float64` for API fields; use `resource.Quantity` when you need portable decimal representations.

## Int-or-string

Use `intstr.IntOrString` when the API naturally allows either:

```go
import "k8s.io/apimachinery/pkg/util/intstr"

type FooSpec struct {
  Port intstr.IntOrString `json:"port"`
}
```

## Object references

For references to other objects, choose one of:

1. Simple string name (most common):

```go
// +kubebuilder:validation:MinLength=1
SecretName string `json:"secretName"`
```

2. Structured reference (name + optional namespace, plus kind/group if needed):

```go
type LocalObjectReference struct {
  // +kubebuilder:validation:MinLength=1
  Name string `json:"name"`
}

type NamespacedObjectReference struct {
  // +kubebuilder:validation:MinLength=1
  Name string `json:"name"`
  Namespace string `json:"namespace,omitempty"`
}
```

If you need to reference arbitrary Kinds, avoid inventing your own schema blindly—read [`../k8s-crd-design-review/references/object-references.md`](../k8s-crd-design-review/references/object-references.md:1).

## Enums and unions (best-effort)

Enums:

```go
// +kubebuilder:validation:Enum=Pending;Ready;Failed
type Phase string

const (
  PhasePending Phase = "Pending"
  PhaseReady   Phase = "Ready"
  PhaseFailed  Phase = "Failed"
)
```

OpenAPI unions (`oneOf`) are limited in CRDs. Prefer explicit structs with clear fields over clever unions.
