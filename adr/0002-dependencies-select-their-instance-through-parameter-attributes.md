---
id: ADR-0002
title: Dependencies select their instance through parameter attributes
status: accepted
created: 2026-03-24
decided: 2026-03-28
backfilled: 2026-09-14
depends: []
updates: []
obsoletes: []
---

# ADR-0002: Dependencies select their instance through parameter attributes

## Context

A dependency's type often does not determine which instance it needs. Several implementations or instances can
share one interface: every database connection is a `Connection`. A parameter typed against that interface says
what it needs, but not which one, so the choice has to be expressed somewhere else.

What is known about the instances varies. Sometimes every concrete, and what maps to it, is known when the panel
boots. Sometimes the concrete is only decided at runtime, from configuration or other state, by the subsystem that
owns the instances.

The panel is dynamic and built for one purpose, and third-party modules consume what its subsystems provide. A
module cannot be expected to know the exact final concrete it needs, especially when configuration can change it.

## Decision

A dependency says which instance it needs with an attribute on its own parameter, and what acts on that attribute
depends on what is known. Where every concrete and its mapping are known at boot, a `Named` attribute maps a string,
or a qualifier attribute maps its class, to a binding registered for the type, and a factory on that binding
produces the instance where construction needs to be dynamic. Where the concrete depends on configuration or other
runtime state, a resolvable attribute is paired with a resolver belonging to the subsystem that owns the instances,
and the resolver decides the instance entirely, with nothing about it known to the bindings. The consuming class
plays no part in the choice, and the correct bindings and resolvers are trusted to exist.

## Alternatives

**Contextual binding by consuming class**, as in Laravel's container, where registration states which concrete a
given consuming class receives for a dependency. It requires knowing the exact final concrete for each consumer,
which a third-party module cannot be expected to know, particularly when configuration can change it. In a system
this dynamic and built for one purpose, it would add a great deal of complication, where the correct bindings and
resolution can instead be trusted to exist.

**Contextual attributes, as in Laravel's container.** Their shape comes from being added to an existing system under
a strict backwards compatibility policy. A contextual attribute is either paired with a resolver closure registered
for it, or carries a `resolve()` method, which receives the attribute instance as a redundant argument. The panel's
author built the original implementation of the idea and co-authored the one Laravel merged, which shipped both
forms together. A resolvable attribute paired with a resolver class is a more structured implementation of the same
idea, without the constraint of fitting an existing system.

No other alternatives were weighed.

## Consequences

Easier:

- A dependency asks for what it needs on its own parameter, and neither its consuming class nor whatever registers
  it has to know the concrete:

  ```php
  public function __construct(
      #[Named('secondary')] private Contract $contract,
      #[Database('replica')] private Connection $connection,
  ) {}
  ```

- Where the mapping is known at boot, it lives in the bindings, as part of registration.
- Where it is not, the subsystem that owns the instances keeps full control over producing them. Its resolver is
  itself resolved through the container, so it receives dependencies of its own, as the database resolver receives
  the connection factory.

Harder:

- Nothing checks, when bindings and resolvers are registered, that every name, qualifier and resolvable attribute in
  use has something to match it. A resolvable attribute with no registered resolver fails when a dependency carrying
  it is resolved.
- A subsystem that decides its instances at runtime defines both an attribute and a resolver, and registers the pair.

Constrained:

- Code selects an instance through these attributes, never by naming a concrete for a consuming class.
- Selection decided at runtime follows the resolver shape. Ghost objects are its first use in the container, and
  later subsystems follow it: the database connection attribute, providers for cycle-scoped instances, injected
  module manifests and registrars, and log channels.

## Sources

- Issue [#21], Engine - Dependency Injection, revision of 2026-03-24 in its edit history: named and qualified
  bindings, factories on bindings, and custom resolvers, the first record of the decision, which is `created`.
- Commit [798470f], "Add resolution and invocation to the container", 2026-03-25: `Resolvable`, `Resolver`,
  `ResolverRegistry` and `ResolverCatalogue`.
- PR [#23], feat(container): Dependency Injection, merged 2026-03-28, with named and qualified bindings and `Ghost`
  and `GhostResolver` as the first resolvable pair: `decided`. When the decision was taken is not recorded.
- Named and qualified bindings being for concretes known at boot, and resolvers for concretes decided by
  configuration or runtime state; resolvers being intended from the start, so that subsystems keep full control over
  their instances; the rejection of contextual binding by consuming class and why; and resolvers as a more structured
  implementation of the contextual attributes whose original implementation the panel's author built and whose
  merged implementation they co-authored, including how that implementation was shaped by backwards compatibility
  and the two forms it shipped with: first written down on 2026-09-14.
- Laravel documentation, Service Container, ["Contextual Binding"][laravel-binding] and
  ["Contextual Attributes"][laravel-attributes], read 2026-09-14: the two Laravel mechanisms described as alternatives.
- PR [#30], feat(database): Add the database component, merged 2026-04-17: the `Database` attribute and
  `DatabaseResolver`, taken from what followed rather than anticipated.
- Issues [#36], [#63] and [#70]: `#[Manifest]` and `#[Registrar]`, `#[Provide]` with `ProviderResolver`, and
  `#[Channel]` composed with module scope, taken from later designs rather than anticipated.

[#21]: https://github.com/thegamepanel/panel/issues/21
[798470f]: https://github.com/thegamepanel/panel/commit/798470f
[#23]: https://github.com/thegamepanel/panel/pull/23
[#30]: https://github.com/thegamepanel/panel/pull/30
[#36]: https://github.com/thegamepanel/panel/issues/36
[#63]: https://github.com/thegamepanel/panel/issues/63
[#70]: https://github.com/thegamepanel/panel/issues/70
[laravel-binding]: https://laravel.com/docs/12.x/container#contextual-binding
[laravel-attributes]: https://laravel.com/docs/12.x/container#contextual-attributes
