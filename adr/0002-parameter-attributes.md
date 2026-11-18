---
id: ADR-0002
title: Select instances with parameter attributes
status: accepted
created: 2026-03-24
decided: 2026-03-28
backfilled: 2026-11-18
depends: []
updates: []
obsoletes: []
---

# ADR-0002: Select instances with parameter attributes

## Context

The type of a [dependency](../GLOSSARY.md#dependency) often does not say which instance it needs. Several
implementations or instances can share one interface. A parameter typed against the interface says what it needs but
not which one, so the choice has to be made somewhere else.

How much is known about the instances varies. Sometimes every [concrete](../GLOSSARY.md#concrete) class, and what maps
to it, is known when the panel boots. Sometimes the concrete is decided only at runtime, from configuration or other
state, by the subsystem that owns the instances.

The panel is dynamic and built for one purpose, and third-party [module](../GLOSSARY.md#module)s consume what its
subsystems provide. A module cannot be expected to know the exact concrete it will receive, least of all when
configuration can change it.

## Decision

I will have a [dependency](../GLOSSARY.md#dependency) say which instance it needs with an attribute on its own
parameter, and what acts on that attribute will depend on what is known. Where every [concrete](../GLOSSARY.md#concrete)
and its mapping are known at boot, a `Named` attribute maps a string, or a [qualifier](../GLOSSARY.md#qualifier)
attribute maps its class, to a [binding](../GLOSSARY.md#binding) registered for the type, and a factory on that binding
produces the instance where construction has to be dynamic. Where the concrete depends on configuration or other runtime
state, a [resolvable attribute](../GLOSSARY.md#resolvable-attribute) is paired with a
[resolver](../GLOSSARY.md#resolver) belonging to the subsystem that owns the instances, and the resolver decides the
instance entirely, with nothing about it known to the bindings. The consuming class plays no part in the choice, and the
correct bindings and resolvers are trusted to exist.

## Alternatives

**Contextual binding by consuming class**, as in Laravel's container, where registration states which
[concrete](../GLOSSARY.md#concrete) class a given consuming class receives for a
[dependency](../GLOSSARY.md#dependency). It would mean knowing the exact concrete for each consumer. I rejected this
because a third-party [module](../GLOSSARY.md#module) cannot be expected to know that, particularly when configuration
can change it, and because in a system this dynamic and built for one purpose it would add a great deal of complication
where the correct [binding](../GLOSSARY.md#binding)s and resolution can instead be trusted to exist.

**Contextual attributes, as in Laravel's container.** A contextual attribute is either paired with a resolver closure
registered for it, or carries a `resolve()` method of its own, which receives the attribute instance as a redundant
argument. The panel's author built the original implementation of the idea and co-authored the one Laravel merged, which
shipped both forms together; their shape comes from being added to an existing system under a strict backwards
compatibility policy. I rejected this because a [resolvable attribute](../GLOSSARY.md#resolvable-attribute) paired with
a [resolver](../GLOSSARY.md#resolver) class is a more structured implementation of the same idea, free of the constraint
of fitting an existing system.

## Consequences

Easier:

- A [dependency](../GLOSSARY.md#dependency) will ask for what it needs on its own parameter, and neither its consuming
  class nor whatever registers it will have to know the [concrete](../GLOSSARY.md#concrete):

  ```php
  public function __construct(
      #[Named('minecraft')] private ServerTemplate $template,
      #[Ghost] private ServerRepository $servers,
  ) {}
  ```

- Where the mapping is known at boot, it will live in the [binding](../GLOSSARY.md#binding)s, as part of registration.
- Where it is not, the subsystem that owns the instances will keep full control over producing them. Its
  [resolver](../GLOSSARY.md#resolver) will itself be resolved through the container, so it can receive dependencies of
  its own.

Harder:

- Nothing will check, when bindings and resolvers are registered, that every name, [qualifier](../GLOSSARY.md#qualifier)
  and [resolvable attribute](../GLOSSARY.md#resolvable-attribute) in use has something to match it. A resolvable
  attribute with no registered resolver will fail when a dependency carrying it is resolved.
- A subsystem that decides its instances at runtime will define both an attribute and a resolver, and register the
  pair.

Constrained:

- Code will select an instance through these attributes, never by naming a concrete for a consuming class.
- Selection decided at runtime will follow the resolver shape, with [ghost object](../GLOSSARY.md#ghost-object)s as its
  first use in the container.

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

[#21]: https://github.com/thegamepanel/panel/issues/21
[798470f]: https://github.com/thegamepanel/panel/commit/798470f
[#23]: https://github.com/thegamepanel/panel/pull/23
[laravel-binding]: https://laravel.com/docs/12.x/container#contextual-binding
[laravel-attributes]: https://laravel.com/docs/12.x/container#contextual-attributes
