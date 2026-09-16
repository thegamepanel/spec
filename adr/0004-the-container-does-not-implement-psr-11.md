---
id: ADR-0004
title: The container does not implement PSR-11
status: accepted
created: 2026-03-25
decided: 2026-03-28
backfilled: 2026-09-14
depends: []
updates: []
obsoletes: []
---

# ADR-0004: The container does not implement PSR-11

## Context

PSR-11 defines a common container interface: `get()` and `has()`, keyed by a string identifier. Its value is
interchangeability. A project can swap one container for another, and a component written against the interface can
be reused with any container that implements it.

The panel is a bespoke, purpose-built system distributed as a single binary, not a framework. Nobody swaps its
container out, and nothing integrates with the panel at the level of its container.

## Decision

The container does not implement PSR-11. Callers describe what they want to resolve or invoke with the container's
own `Resolution` and `Invocation` objects, not with a string identifier passed to `get()`, and no part of the panel
depends on the `Psr\Container` interfaces.

## Alternatives

**Implementing PSR-11**, alongside the container's own API. The interchangeability it provides does not apply to a
bespoke system shipped as a single binary. The PSRs are also dated, and implementing one forces a design into its
shape, where the container is instead free to be shaped around what the panel needs.

No other alternative was weighed.

## Consequences

Easier:

- The container's API follows the panel's needs. A resolution carries its name, qualifier, arguments, and whether
  it is lazy or liminal as part of the request, rather than encoding them in an identifier.

Harder:

- A library that expects a PSR-11 container cannot be given the panel's container.

Constrained:

- Components depend on the container's own API, and on nothing from PSR-11.

## Sources

- Commit [bdc32d7], "Initial dependency injection work", 2026-03-25: the first code of the container, without
  PSR-11, which is `created`.
- PR [#23], feat(container): Dependency Injection, merged 2026-03-28: the container merged without PSR-11, which is
  `decided`. When the decision was taken is not recorded.
- The decision and its reasons, that PSR-11's interchangeability does not apply to a bespoke system shipped as a
  single binary and that PSRs are dated and constrain design: first written down on 2026-09-14.
- Issue [#31], Engine - Config - TOML, original text of 2026-03-30 in its edit history: the first written record of
  the panel being distributed as a binary, two days after this decision's date.
- [`composer.json` at 728ad64](https://github.com/thegamepanel/panel/blob/728ad64/composer.json): no dependency on
  `psr/container`.

[bdc32d7]: https://github.com/thegamepanel/panel/commit/bdc32d7
[#23]: https://github.com/thegamepanel/panel/pull/23
[#31]: https://github.com/thegamepanel/panel/issues/31
