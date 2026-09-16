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

Changes to how the container caches shared instances and resolves classes. Shared instances are kept in one instance
cache type, held twice for strong and weak retention. Class-level marker attributes are memoised as presence flags,
and whether a resolution is liminal is worked out from every source before the cache is checked. Circular
dependencies are detected, and reported with the chain of classes involved.

## Motivation

The container keeps shared instances in four separate caches, and the logic that picks one is written twice, once
for reading and once for writing, so the two can disagree.

When a resolution is checked against the cache, only what the resolution itself says is known. Whether its binding
or its class makes it liminal is found out afterwards, once the cache has already been consulted.

A class that depends on itself, directly or through other classes, exhausts the call stack, and nothing reports
which classes are involved.

## Proposal

### Concepts

| Term | Meaning |
|---|---|
| Instance cache | A store of shared instances, keyed by class, and by name or qualifier class where a resolution has one. |
| Retention | Whether an instance cache holds its instances strongly, or weakly through weak references. |
| Effective liminality | Whether a resolution is liminal, worked out from the resolution, its binding and its class. |
| Resolution stack | The resolutions being constructed eagerly, in the order resolution reached them. |
| Circular dependency | A resolution whose eager construction requires the same resolution again, directly or through others. |

### Components

| Component | Responsibility |
|---|---|
| `InstanceCache` | Holds shared instances under a class, and a name or qualifier class, with strong or weak retention. |
| `Container` | Holds a strong and a weak instance cache, the memoised class attribute flags, and the resolution stack. |
| `BindingCatalogue` | Exposes alias normalisation, so the container can normalise a class before consulting the cache. |
| `CircularDependencyException` | Thrown when a circular dependency is found. |

### Instance cache

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

A cache holds one instance per class, one per class and name, and one per class and qualifier class. The qualifier is
its class, so a qualified instance is found by a direct lookup, as the qualifier's class is the key in
[RFC-0001](0001-dependency-injection-container.md).

Retention is chosen when the cache is created. Under weak retention the weak reference is internal: `get()` returns
the instance, typed as the class requested, or `null`, both when nothing was stored and when the stored instance has
been collected.

The cache knows nothing of bindings or aliases. It stores and returns under whatever class it is given, and the
container normalises the class first.

The container holds two caches:

```php
private InstanceCache $instances        = InstanceCache::strong();
private InstanceCache $liminalInstances = InstanceCache::weak();
```

Reading and writing both pick a cache by effective liminality, then delegate to it, so the choice is made in one
place. A liminal resolution and a shared resolution of the same class occupy separate entries.

### Class attributes

The container memoises, for each class, whether it carries `NoResolution`, `Lazy` and `Liminal`, per
[ADR-0014](../adr/0014-class-level-attributes-are-memoised-as-presence-flags.md). The flags come from one unfiltered
`getAttributes()` pass matched on `getName()` the first time the class is resolved, and no attribute is instantiated.
Later resolutions of the class read the flags without reflection.

Nothing is memoised about parameters, per
[ADR-0020](../adr/0020-resolution-data-is-not-memoised-per-parameter.md).

### Resolution

These steps replace the resolution steps of [RFC-0001](0001-dependency-injection-container.md):

1. **Alias.** An alias is replaced by the abstract it points to.
2. **Binding.** The binding for the class, name and qualifier is looked up.
3. **Liminality.** The resolution is liminal if it asks to be, if its binding marks it liminal, or if the class
   carries `Liminal`, read from the memoised flags. This is worked out once, and used for both reading and writing
   the cache.
4. **Cached instance.** If a shared instance is cached, it is returned: from the weak cache for a liminal
   resolution, while the instance is still alive, and from the strong cache otherwise, under the name or qualifier
   class where the resolution has one.
5. **Lazy proxy.** If the resolution is lazy, a lazy proxy is returned, and the steps below run when it is first
   used.
6. **Instance.** If the binding holds an instance, that instance is the result. If it has a factory, the factory is
   invoked through the container and its return value is the result. If it has a concrete class, that class is
   constructed in place of the abstract.
7. **Automatic construction.** Otherwise the class is constructed by the container. A class marked `NoResolution`,
   read from the memoised flags, cannot be, and throws. A class that is not instantiable throws. A class with no
   constructor is instantiated directly, and a class with one has its constructor invoked through the container.
8. **Class attributes.** A class marked `Lazy`, read from the memoised flags, is always resolved as a lazy proxy.
9. **Sharing.** Unless a binding marks it not shared, the instance is cached: in the weak cache if the resolution is
   liminal, and in the strong cache otherwise, under the class from the first step, and the name or qualifier class
   where the resolution has one.

### Circular dependencies

The container holds a resolution stack. Each resolution adds an entry when it starts, and removes it when it
finishes, whether it returns or throws. An entry is identified the same way as a shared instance in the instance
cache: by the class after alias normalisation, and the name or qualifier class where the resolution has one.

A resolution whose entry is already on the stack throws `CircularDependencyException`. Its message names every class
in the chain in order, as each was requested, from the first appearance of the entry to its repetition, such as `A`,
`B`, `C` and then `A` again.

Only eager construction of the same resolution is detected:

- A class that depends on a named or qualified child binding of its own abstract is not a cycle, because the two
  resolutions have different entries. A decorator bound to `Cache`, whose constructor takes the `Cache` binding named
  `inner`, resolves normally.
- A lazy proxy or a ghost is returned before its constructor runs, and its resolution leaves the stack as it
  returns, so `Lazy` and `Ghost` still break a cycle.
- A class resolved twice in sequence, such as the same class for two parameters of one constructor, is not a cycle,
  because the first resolution has left the stack before the second starts.
- A resolution that throws leaves the stack as it was before it started, so an unrelated resolution afterwards is
  not reported as a cycle.

### Errors

| Exception | Extends | Thrown when |
|---|---|---|
| `CircularDependencyException` | `RuntimeException` | A class being constructed eagerly is required again before its construction finishes. It carries the chain of classes in order, and implements `ContainerException`. |

### Out of scope

- **Chained aliases.** An alias pointing to another alias is not followed. Whether chains are flattened when
  catalogues are built, or followed at runtime, belongs to building catalogues.
- **Memoising parameters and resolution plans**, ruled out by
  [ADR-0020](../adr/0020-resolution-data-is-not-memoised-per-parameter.md).

## Alternatives considered

The decisions this design rests on are recorded separately, with the alternatives each one rejected:

- memoising class-level attributes as presence flags, in
  [ADR-0014](../adr/0014-class-level-attributes-are-memoised-as-presence-flags.md)
- not memoising resolution data per parameter, in
  [ADR-0020](../adr/0020-resolution-data-is-not-memoised-per-parameter.md)

**One cache holding every instance through weak references.** A weak reference is pointless beside a strong one in
the same store, and a liminal and a shared resolution of one class are different cache identities: resolving a class
without liminality should still cache it normally.

**Choosing retention with an enum or a flag.** Static factories follow the container's own convention, such as
`Resolution::for()` and `Invocation::callable()`.

No other alternatives were weighed.

## Backwards compatibility

Nothing that resolves today breaks. A circular dependency throws `CircularDependencyException` instead of exhausting
the call stack.

## Open questions

- **Collected weak entries.** An entry whose instance has been collected stays in the weak cache, and `get()` returns
  `null` for it. Whether such entries are removed when a read finds them, or kept and documented, is not settled.

## Changelog

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
