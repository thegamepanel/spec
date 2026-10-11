---
id: ADR-0029
title: Require a binding for named and qualified resolutions
status: proposed
created: 2026-10-08
decided:
depends: [ADR-0002, RFC-0001, RFC-0006]
updates: []
obsoletes: []
---

# ADR-0029: Require a binding for named and qualified resolutions

## Context

A [binding](../GLOSSARY.md#binding) says how an [abstract](../GLOSSARY.md#abstract) is produced. Where one abstract is
produced more than one way, such as a `Cache` with a decorator bound to it and the cache it decorates bound beneath it
under the name `inner`, each further way is a [named binding](../GLOSSARY.md#named-binding) or a
[qualified binding](../GLOSSARY.md#qualified-binding) beneath the abstract's own. A
[dependency](../GLOSSARY.md#dependency) selects one through the attribute on its parameter, as
[ADR-0002](0002-dependencies-select-their-instance-through-parameter-attributes.md) decides.

A [resolution](../GLOSSARY.md#resolution) carries a name or a [qualifier](../GLOSSARY.md#qualifier), never both. In the
resolution steps of [RFC-0006](../rfc/0006-container-improvements.md), the binding lookup for a resolution with a name
or qualifier finds the child binding under that name or qualifier class, or nothing: nothing when the abstract has no
binding at all, and nothing when its binding has no child under that key. Where it finds nothing, the construction step
constructs the requested class, and the sharing step caches the instance under the name or qualifier class. The name or
qualifier then selects nothing, the instance is the requested class under a key that no binding defines, and a mistyped
or unregistered name is never reported.

[RFC-0001](../rfc/0001-dependency-injection-container.md) defines `BindingNotFoundException` and lists no case that
throws it.

## Decision

We will treat a name or [qualifier](../GLOSSARY.md#qualifier) as selecting among the bindings of one
[abstract](../GLOSSARY.md#abstract). A [resolution](../GLOSSARY.md#resolution) that carries a name or qualifier and
whose lookup finds no [binding](../GLOSSARY.md#binding), because the abstract has no binding or because its binding has
no child under that name or qualifier class, will throw `BindingNotFoundException` at the lookup, naming the class and
the name or qualifier class, and will construct nothing. A resolution without a name or qualifier is unchanged.

## Alternatives

**Constructing the requested class**, as [RFC-0006](../rfc/0006-container-improvements.md) specifies. The name or
qualifier would find no [binding](../GLOSSARY.md#binding) and still key the cache, so the
[resolution](../GLOSSARY.md#resolution) would produce the requested class under that key. We rejected this because a
name or [qualifier](../GLOSSARY.md#qualifier) that selects nothing would go unnoticed, so a mistyped or unregistered
name would surface only when an instance behaved differently from the one intended.

**Falling back to the abstract's own binding.** A resolution whose [abstract](../GLOSSARY.md#abstract) has a binding
with no child under the name or qualifier class would use the abstract's own binding. We rejected this because a name or
qualifier asks for something other than the abstract's own binding, so producing it would hide a mistyped or
unregistered name in the same way.

## Consequences

Easier:

- A name or [qualifier](../GLOSSARY.md#qualifier) that matches no [binding](../GLOSSARY.md#binding) will fail at the
  [resolution](../GLOSSARY.md#resolution) that asked for it, with an exception naming the class and the name or
  qualifier class.
- The [instance cache](../GLOSSARY.md#instance-cache) will hold named and qualified entries only for bindings that
  exist.

Harder:

- Every name and qualifier a [dependency](../GLOSSARY.md#dependency) carries will need a binding registered before it is
  resolved.

Constrained:

- A name or qualifier will never select an instance of a class that has no binding. Selecting an instance by anything a
  binding does not know, such as a value, will go through a [resolvable attribute](../GLOSSARY.md#resolvable-attribute)
  and its [resolver](../GLOSSARY.md#resolver), as
  [ADR-0002](0002-dependencies-select-their-instance-through-parameter-attributes.md) decides.
