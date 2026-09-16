---
id: ADR-0014
title: Class-level attributes are memoised as presence flags
status: accepted
created: 2026-08-08
decided: 2026-08-08
backfilled: 2026-09-14
depends: [RFC-0001]
updates: []
obsoletes: []
---

# ADR-0014: Class-level attributes are memoised as presence flags

## Context

The container consults its instance cache before it reflects on a class, so it cannot tell whether a class is
liminal at the point it checks the cache. A class carrying `#[Liminal]` is stored in the liminal cache but looked up
in the ordinary one, and never produces a cache hit. Fixing that requires the cache check to know class-level
liminality, and reflecting on every resolution to find out would defeat the purpose of checking the cache first.

The three class-level attributes from [RFC-0001](../rfc/0001-dependency-injection-container.md), `NoResolution`,
`Lazy` and `Liminal`, are markers. Resolution only tests them for presence and never reads a property from any of
them, so no instance is needed. `ReflectionAttribute::getName()` does not instantiate the attribute, which a probe
attribute with a counting constructor confirms. That matters once the module system lands and third-party classes
carry attributes from unrelated libraries that the container has no business constructing.

The change is a correctness prerequisite for fixing liminal cache misses, not a performance change.

## Decision

The container memoises class-level attribute lookups for `NoResolution`, `Lazy` and `Liminal` as a map of booleans
keyed by class-string, populated by a single unfiltered `getAttributes()` pass matched on `getName()`. No attribute
is instantiated, and no reflection happens on the cache-check path after the first resolution of a class.

## Alternatives

- **Caching every attribute instance eagerly.** `newInstance()` runs user code, and third-party classes will carry
  attributes the container never asked about, which can throw and can be slow.
- **Caching instances as they are requested.** The cold path still costs one filtered reflection call per attribute
  per class, and instances are retained that markers do not need.
- **Caching `ReflectionClass` instances.** It saves about 0.4 µs from a 98.6 µs resolution, roughly 0.4%, because
  parsed class metadata is already shared and the reflector is a thin handle over it. It is written down here so
  that it is not revisited.

Measured on a `Root(Mid, Leaf, Mid)` and `Mid(Leaf, Leaf)` graph of 8 resolve calls and 7 constructor parameters:

| Operation | Cost |
|---|---|
| Three filtered `getAttributeInstance()` calls per resolution, before this decision | 1.750 µs |
| One unfiltered scan matched on `getName()` | 0.692 µs |
| A memoised array hit | 0.277 µs |

## Consequences

Easier:

- Class-level liminality will be available when the cache is checked, which the fix for liminal cache misses needs.
- Attribute constructors from unrelated libraries will never run during resolution.

Harder:

- Parameter-level attribute handling in `createDependency()` will remain the actual hot path, at about 3 µs per
  constructor parameter with 64% of that in its four `getAttributeInstance()` calls. Caching the whole resolution
  plan measures about fifteen times cheaper there, cutting roughly 20% from a representative resolution, and is left
  to be considered separately.

Constrained:

- Only attributes that resolution tests for presence can be memoised this way. An attribute whose properties
  resolution reads needs an instance, so the technique will not extend to attributes that carry values.

## Sources

- Issue [#43], Engine - Container - Memoise class-level attribute lookups, 2026-08-08: the problem, the decision,
  the three alternatives, the measurements, memoisation relying on the attributes being markers, and the
  parameter-level follow-up. It is both `created` and `decided`.
- Issue [#46], Engine - Container - Liminal information parity, 2026-08-08: the liminal cache misses this decision is
  a prerequisite for.

[#43]: https://github.com/thegamepanel/panel/issues/43
[#46]: https://github.com/thegamepanel/panel/issues/46
