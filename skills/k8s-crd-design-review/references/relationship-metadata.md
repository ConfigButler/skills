# CRD relationship metadata (exploratory proposal)

> **Status:** An intentionally small proposal for tool authors. It is not a
> Kubernetes, Flux, or controller-runtime standard.

## Problem

A CRD schema can describe the shape of `spec.gitProviderRef.name`, but it does
not machine-readably say that the value resolves to a namespaced
`gitops.configbutler.io` `gitproviders` resource. Editors, graph tools, and
GitOps-aware refactorers therefore need project-specific knowledge to provide
"go to definition", find usages, safe rename, and dependency-graph features.

Putting a defaulted `group` and `kind` into every fixed-kind reference is not a
solution. Those values are not a choice made by the manifest author, enlarge
the public API, and falsely imply that the reference is polymorphic.

## Design principles

- **Describe; do not control.** Metadata helps tools discover relationships. It
  does not grant access, validate target existence, decide reconcile order, or
  implement cross-namespace authorization.
- **Keep author intent in the instance.** A fixed-kind reference can remain
  `{name}`. A bounded multi-kind reference keeps `kind` because selecting the
  kind is real intent.
- **Use Group/Resource for targets.** A resource is the canonical API endpoint;
  Kind is safe only where a controller owns a predefined mapping.
- **Declare namespace semantics.** Same-namespace, explicit-namespace, and
  cluster-scoped targets must be distinguishable. Cross-namespace authorization
  remains a separate concern, for example a `ReferenceGrant`-style opt-in.
- **Version the metadata separately.** The metadata must evolve and survive CRD
  version conversion without forcing a change to stored custom resources.

## Minimal relationship model

The conceptual model contains a referrer, a field path, an identity shape, one
or more possible targets, and namespace semantics.

```yaml
version: v1alpha1
references:
  - referrer:
      group: gitops.configbutler.io
      version: v1alpha3
      kind: GitTarget
      path: .spec.gitProviderRef
    identity:
      namePath: .name
    targets:
      - group: gitops.configbutler.io
        resource: gitproviders
    namespace:
      mode: SameNamespace
```

The target has no version: the controller and tooling resolve a served version
of the declared Group/Resource. A tool may obtain the display Kind through
discovery or the target CRD; it must not infer the resource from an arbitrary
Kind.

### Bounded multi-kind example

For a Flux-style `sourceRef`, the selector is part of the user-authored
reference and maps to a bounded, implementation-owned target set:

```yaml
version: v1alpha1
references:
  - referrer:
      group: example.io
      version: v1
      kind: Delivery
      path: .spec.sourceRef
    identity:
      namePath: .name
      selectorPath: .kind
    targets:
      - selectorValue: GitRepository
        group: source.toolkit.fluxcd.io
        resource: gitrepositories
      - selectorValue: OCIRepository
        group: source.toolkit.fluxcd.io
        resource: ocirepositories
    namespace:
      mode: SameNamespaceOrField
      path: .namespace
```

The exact names are illustrative. A standard must precisely define the valid
namespace modes and field-path syntax before declaring this wire format stable.

## Delivery path

### 1. Prototype outside the CRD schema

Start with the versioned model in a CRD metadata annotation or a plugin registry
owned by the API provider. For example:

```yaml
metadata:
  annotations:
    relationships.configbutler.io/v1alpha1: |-
      references:
        - referrer:
            version: v1alpha3
            kind: GitTarget
            path: .spec.gitProviderRef
          identity: {namePath: .name}
          targets:
            - group: gitops.configbutler.io
              resource: gitproviders
          namespace: {mode: SameNamespace}
```

This is an experimental transport, not a claim on a Kubernetes-owned annotation
namespace. It lets an editor prototype prove navigation and rename behavior
without changing resource instances or requiring API-server support.

### 2. Generate the same model from API-owned declarations

Avoid a hand-maintained map. A controller-gen extension or a small companion
generator should emit the metadata from field-level declarations in the API
types, then test that the metadata and generated CRD stay synchronized.

### 3. Propose a Kubernetes schema extension only after adoption

The eventual form could be a property-level `x-kubernetes-reference` extension:

```yaml
gitProviderRef:
  type: object
  required: [name]
  properties:
    name:
      type: string
      minLength: 1
  x-kubernetes-reference:
    targets:
      - group: gitops.configbutler.io
        resource: gitproviders
    namespace:
      mode: SameNamespace
```

This extension does not exist today. It would need an API-machinery proposal
that specifies validation, preservation through conversion, and publication in
OpenAPI. Consumers that do not understand it must be able to ignore it safely.

## Non-goals

- Replacing the upstream object-reference shapes.
- Adding Group/Kind/Version values to fixed-kind resource instances.
- Verifying that a target exists or is ready.
- Authorizing cross-namespace references; use a target-owner opt-in such as
  Gateway API `ReferenceGrant` where that is required.
- Expressing controller ownership (`ownerReferences`) or orchestration
  (`dependsOn`).

## Why Flux is a useful design partner

Flux already distinguishes the two key cases. Its single-purpose references,
such as a Secret reference, use only a name. Its `Kustomization.spec.sourceRef`
uses `kind`, `name`, and an optional namespace because the caller selects from
a documented set of source types. Relationship metadata could tell tools about
that mapping without changing Flux manifests or controller behavior.

The proposal should therefore ask Flux to review a low-cost, generated metadata
contract—not to adopt a new reference object shape or a ConfigButler-specific
registry. A useful first interoperability test is whether one plugin can follow
a ConfigButler fixed-kind reference and a Flux bounded source reference from
their installed CRDs alone.

## Related work and source links

- [Kubernetes API conventions: Object references](https://github.com/kubernetes/community/blob/main/contributors/devel/sig-architecture/api-conventions.md)
  define the single-, multi-, generic-, and field-reference models this
  proposal preserves.
- [Flux Kustomization source references](https://fluxcd.io/flux/components/kustomize/kustomizations/#source-reference)
  document the bounded `kind` choice and optional cross-namespace field.
- [Flux Kustomize API reference](https://fluxcd.io/flux/components/kustomize/api/v1/)
  exposes `CrossNamespaceSourceReference` as a typed API contract.
- [Historical Kustomize schema hints discussion](https://github.com/kubernetes-sigs/kustomize/issues/4095)
  covers `x-kubernetes-object-ref-*` hints and why they did not become a shared
  SIG-tooling contract.
- [Kubernetes CRD API reference](https://kubernetes.io/docs/reference/kubernetes-api/apiextensions/custom-resource-definition-v1/)
  lists the supported schema extensions available today.
- [Kubernetes API union extension KEP](https://github.com/kubernetes/enhancements/blob/master/keps/sig-api-machinery/1027-api-unions/README.md)
  is a useful precedent for defining a new `x-kubernetes-*` extension and
  preserving it through OpenAPI conversion.
- [Gateway API ReferenceGrant](https://gateway-api.sigs.k8s.io/api-types/referencegrant/)
  is a precedent for making cross-namespace reference authorization explicit.
