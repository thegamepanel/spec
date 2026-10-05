---
id: RFC-0006
title: Container improvements
status: proposed
created: 2026-09-14
decided:
depends: [ADR-0014, ADR-0020]
updates: [RFC-0001]
obsoletes: []
---

# RFC-0006: Container improvements

## Abstract

We will change how the container caches [shared](../GLOSSARY.md#shared) instances and resolves classes. Shared instances
will be kept in one [instance cache](../GLOSSARY.md#instance-cache) type, used once with strong references and once with
weak references. Class-level marker attributes will be memoised as presence flags, and whether a
[resolution](../GLOSSARY.md#resolution) is [liminal](../GLOSSARY.md#liminal) will be worked out from every source before
the cache is checked. Circular dependencies will be detected, and reported with the chain of classes involved.

## Motivation

The container keeps [shared](../GLOSSARY.md#shared) instances in four separate caches, and the logic that picks one is
written twice, once for reading and once for writing, so the two can disagree.

When a [resolution](../GLOSSARY.md#resolution) is checked against the cache, only what the resolution itself says is
known. Whether its [binding](../GLOSSARY.md#binding) or its class makes it [liminal](../GLOSSARY.md#liminal) is found
out afterwards, once the cache has already been consulted.

A class that depends on itself, directly or through other classes, exhausts the call stack, and nothing reports
which classes are involved.

## Proposal

### Instance cache

We will add `InstanceCache`, a store of [shared](../GLOSSARY.md#shared) instances keyed by class, and by name or
[qualifier](../GLOSSARY.md#qualifier) class where a [resolution](../GLOSSARY.md#resolution) has one. Each cache will
hold its instances either strongly or through weak references.

```php
final class InstanceCache
{
    public static function strong(): self;
    public static function weak(): self;

    public function get(string $class, ?string $name = null, ?string $qualifier = null): ?object;
    public function put(string $class, object $instance, ?string $name = null, ?string $qualifier = null): void;
}
```

| Method | Effect |
|---|---|
| `strong()` | Static. Creates a cache that holds its instances strongly. |
| `weak()` | Static. Creates a cache that holds its instances through weak references. |
| `get(string $class, ?string $name = null, ?string $qualifier = null): ?object` | Returns the instance stored under the class, and the name or qualifier class where one is given, or `null`. |
| `put(string $class, object $instance, ?string $name = null, ?string $qualifier = null): void` | Stores the instance under the class, and the name or qualifier class where one is given. |

A cache will hold one instance per class, one per class and name, and one per class and qualifier class. The qualifier's
class is its key, as it is in [RFC-0001](0001-dependency-injection-container.md), so a qualified instance will be found
by a direct lookup.

Whether a cache holds its instances strongly or weakly will be chosen when it is created. In a weak cache the weak
reference is internal: `get()` will return the instance, typed as the class requested, or `null`, both when nothing was
stored and when the stored instance has been collected.

The cache will know nothing of a [binding](../GLOSSARY.md#binding) or an [alias](../GLOSSARY.md#alias). It will store
and return under whatever class it is given, and the container will normalise the class first. `BindingCatalogue` will
expose alias normalisation for the container to use before it consults the cache.

The container will hold two caches:

```php
private InstanceCache $instances        = InstanceCache::strong();
private InstanceCache $liminalInstances = InstanceCache::weak();
```

Reading and writing will both pick a cache by [effective liminality](../GLOSSARY.md#effective-liminality), then delegate
to it, so the choice is made in one place. A [liminal](../GLOSSARY.md#liminal) resolution and a shared resolution of the
same class will occupy separate entries.

### Class attributes

The container will memoise, for each class, whether it carries `NoResolution`, `Lazy` and `Liminal`, as
[ADR-0014](../adr/0014-class-level-attributes-are-memoised-as-presence-flags.md) decides. The flags will come from one
unfiltered `getAttributes()` pass, matched on `getName()`, the first time the class is resolved, and no attribute will
be instantiated. Later resolutions of the class will read the flags without reflection.

Nothing will be memoised about parameters, as
[ADR-0020](../adr/0020-resolution-data-is-not-memoised-per-parameter.md) decides.

### Resolution

These steps will replace the [resolution](../GLOSSARY.md#resolution) steps of
[RFC-0001](0001-dependency-injection-container.md):

1. **Alias.** An [alias](../GLOSSARY.md#alias) is replaced by the [abstract](../GLOSSARY.md#abstract) it points to.
2. **Binding.** The [binding](../GLOSSARY.md#binding) for the class, name and [qualifier](../GLOSSARY.md#qualifier) is
   looked up.
3. **Liminality.** The resolution is [liminal](../GLOSSARY.md#liminal) if it asks to be, if its binding marks it
   liminal, or if the class carries `Liminal`, read from the memoised flags. This is worked out once, and used for both
   reading and writing the cache.
4. **Cached instance.** If a [shared](../GLOSSARY.md#shared) instance is cached, it is returned: from the weak cache for
   a liminal resolution, while the instance is still alive, and from the strong cache otherwise, under the name or
   qualifier class where the resolution has one.
5. **Lazy proxy.** If the resolution is lazy, a [lazy proxy](../GLOSSARY.md#lazy-proxy) is returned, and the steps below
   run when it is first used.
6. **Instance.** If the binding holds an instance, that instance is the result. If it has a factory, the factory is
   invoked through the container and its return value is the result. If it has a [concrete](../GLOSSARY.md#concrete)
   class, that class is constructed in place of the abstract.
7. **Automatic construction.** Otherwise the class is constructed by the container. A class marked `NoResolution`,
   read from the memoised flags, cannot be, and throws. A class that is not instantiable throws. A class with no
   constructor is instantiated directly, and a class with one has its constructor invoked through the container.
8. **Class attributes.** A class marked `Lazy`, read from the memoised flags, is always resolved as a lazy proxy.
9. **Sharing.** Unless a binding marks it not shared, the instance is cached: in the weak cache if the resolution is
   liminal, and in the strong cache otherwise, under the class from the first step, and the name or qualifier class
   where the resolution has one.

### Circular dependencies

A circular dependency is a [resolution](../GLOSSARY.md#resolution) whose eager construction requires the same resolution
again, directly or through others.

The container will hold a [resolution stack](../GLOSSARY.md#resolution-stack). Each resolution will add an entry when it
starts, and remove it when it finishes, whether it returns or throws. An entry will be identified the same way as a
[shared](../GLOSSARY.md#shared) instance in the [instance cache](../GLOSSARY.md#instance-cache): by the class after
[alias](../GLOSSARY.md#alias) normalisation, and the name or [qualifier](../GLOSSARY.md#qualifier) class where the
resolution has one.

A resolution whose entry is already on the stack will throw `CircularDependencyException`. Its message will name every
class in the chain in order, as each was requested, from the first appearance of the entry to its repetition, such as
`A`, `B`, `C` and then `A` again.

Only eager construction of the same resolution will be detected:

- A class that depends on a [named binding](../GLOSSARY.md#named-binding) or
  [qualified binding](../GLOSSARY.md#qualified-binding) of its own [abstract](../GLOSSARY.md#abstract) is not a cycle,
  because the two resolutions have different entries. A decorator bound to `Cache`, whose constructor takes the `Cache`
  binding named `inner`, resolves normally.
- A [lazy proxy](../GLOSSARY.md#lazy-proxy) or a [ghost object](../GLOSSARY.md#ghost-object) is returned before its
  constructor runs, and its resolution leaves the stack as it returns, so `Lazy` and `Ghost` still break a cycle.
- A class resolved twice in sequence, such as the same class for two parameters of one constructor, is not a cycle,
  because the first resolution has left the stack before the second starts.
- A resolution that throws leaves the stack as it was before it started, so an unrelated resolution afterwards is
  not reported as a cycle.

### Errors

A circular dependency will throw `CircularDependencyException`, which extends `RuntimeException` and implements
`ContainerException`. It will be thrown when a class being constructed eagerly is required again before its construction
finishes, and will carry the chain of classes in order.

### Out of scope

- **Chained aliases.** An [alias](../GLOSSARY.md#alias) pointing to another alias is not followed. Whether chains are
  flattened when a [catalogue](../GLOSSARY.md#catalogue) is built, or followed at runtime, belongs to building
  catalogues.
- **Memoising parameters and resolution plans**, ruled out by
  [ADR-0020](../adr/0020-resolution-data-is-not-memoised-per-parameter.md).

## Alternatives considered

The decisions this design rests on are recorded separately, with the alternatives each one rejected:

- memoising class-level attributes as presence flags, in
  [ADR-0014](../adr/0014-class-level-attributes-are-memoised-as-presence-flags.md)
- not memoising resolution data per parameter, in
  [ADR-0020](../adr/0020-resolution-data-is-not-memoised-per-parameter.md)

**One cache holding every instance through weak references.** We rejected this because a weak reference is pointless
beside a strong one in the same store, and a [liminal](../GLOSSARY.md#liminal) and a [shared](../GLOSSARY.md#shared)
resolution of one class are different cache identities: resolving a class without liminality should still cache it
normally.

**Choosing strong or weak references with an enum or a flag.** We rejected this because static factories follow the
container's own convention, such as `Resolution::for()` and `Invocation::callable()`.

No other alternatives were weighed.

## Backwards compatibility

Nothing that resolves today will break. A circular dependency will throw `CircularDependencyException` instead of
exhausting the call stack.

## Open questions

- **Collected weak entries.** An entry whose instance has been collected stays in the weak cache, and `get()` returns
  `null` for it. Whether such entries are removed when a read finds them, or kept and documented, is not settled.

## Changelog

- 2026-10-05: Reworded in the readable house style, and the project's own vocabulary linked to the glossary. The
  design is unchanged.

## Sources

- Issue [#43], Engine - Container - Memoise class-level attribute lookups, 2026-08-08: memoising class-level
  attributes, and making class-level liminality available when the cache is checked.
- Issue [#44], Engine - Container - Alias cache key mismatch, 2026-08-08, edited the same day to land after [#45]:
  normalising aliases once before the cache is consulted, exposing alias normalisation on `BindingCatalogue`, and
  chained aliases left to building catalogues.
- Issue [#45], Engine - Container - Introduce InstanceCache, 2026-08-08: the instance cache, its strong and weak
  factories, its class, name and qualifier dimensions, keeping the weak reference internal, two caches rather than
  weak references throughout, static factories rather than an enum or a flag, alias normalisation staying in the
  container, and collected weak entries left undecided.
- Issue [#46], Engine - Container - Liminal information parity, 2026-08-08: looking the binding up before the cache
  check, and working out liminality once from the resolution, the binding and the class attribute for both reading
  and writing.
- Issue [#47], Engine - Container - Circular dependency detection, 2026-08-08: the resolution stack, pushed on entry
  and removed however the resolution finishes, `CircularDependencyException` carrying the chain in order, lazy
  proxies and ghosts still breaking cycles, and sequential resolution of one class not counting as a cycle.
- The instance cache keying qualified instances by qualifier class, and resolution stack entries being identified the
  same way as instance cache entries, so that named and qualified child bindings of one abstract are not reported as
  cycles: first written down on 2026-09-14.

[#43]: https://github.com/thegamepanel/panel/issues/43
[#44]: https://github.com/thegamepanel/panel/issues/44
[#45]: https://github.com/thegamepanel/panel/issues/45
[#46]: https://github.com/thegamepanel/panel/issues/46
[#47]: https://github.com/thegamepanel/panel/issues/47
