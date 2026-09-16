---
id: ADR-0005
title: Configuration is held in typed objects
status: accepted
created: 2026-03-25
decided: 2026-03-29
backfilled: 2026-09-14
depends: []
updates: []
obsoletes: []
---

# ADR-0005: Configuration is held in typed objects

## Context

The engine's components need configuration, and so do modules. The code that consumes a configuration needs to rely
on its shape and on the types of its values.

## Decision

Each configuration is an instance of a class unique to that configuration, registered against a module and a name.
Its values are typed properties, and it may contain child objects, so the shape and types of a configuration are
enforced by its class rather than by the code that reads it. A configuration object is found by its module and name,
or by its class, and a component receives the one it needs by its class.

## Alternatives

**Configuration as arrays read by key.** Nothing enforces the shape of an array or the types of its values, so it
gives no type safety.

No other alternatives were weighed.

## Consequences

Easier:

- Code reading configuration works with typed values that static analysis can check, rather than looking keys up
  and casting what it finds.
- Because each configuration has a class of its own, a component can be given its configuration by type.

Harder:

- Every configuration needs a class.
- Configuration stored in a file format has to be hydrated into its class, and fails if it cannot be.

Constrained:

- Components and modules define their configuration as classes, registered against a module and a name.

## Sources

- Issue [#22], Engine - Config, 2026-03-25 and not edited since: configuration as instances of classes unique to each
  configuration, registered against a name and a module, and injected because each is a unique class. This is
  `created`.
- PR [#25], feat(config): Add config component, merged 2026-03-29 and squashed as [180794a]: `ConfigObject` and
  `ConfigCatalogue`, with lookup by module and name or by class. This is `decided`.
- Type safety as the reason for the decision: first written down on 2026-09-14.
- Issue [#31], Engine - Config - TOML, revision of 2026-05-19 in its edit history: hydration from a file format that
  fails when a section cannot hydrate, taken from what followed rather than anticipated.

[#22]: https://github.com/thegamepanel/panel/issues/22
[#25]: https://github.com/thegamepanel/panel/pull/25
[#31]: https://github.com/thegamepanel/panel/issues/31
[180794a]: https://github.com/thegamepanel/panel/commit/180794a
