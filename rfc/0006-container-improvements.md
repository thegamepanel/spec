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

A shared instance resolved through an [alias](../GLOSSARY.md#alias) is cached under a different class from the one it
is read under, so each resolution through the alias constructs a new instance. A resolution given both a name and a
[qualifier](../GLOSSARY.md#qualifier) ignores the qualifier without saying so. A
[lazy proxy](../GLOSSARY.md#lazy-proxy) is made for the class as requested, which PHP cannot do for an interface.

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
class is its key, as it is in [RFC-0001](0001-dependency-injection-container.md), and any value the qualifier carries is
not part of it. The container will never give a cache both a name and a qualifier class, since a resolution cannot have
both.

Whether a cache holds its instances strongly or weakly will be chosen when it is created. In a weak cache the weak
reference is internal: `get()` will return the instance, typed as the class requested, or `null`, both when nothing was
stored and when the stored instance has been collected.

An entry whose instance has been collected will stay in a weak cache until the next `put()` under the same class, and
the same name or qualifier class, replaces it. Neither a read that finds it nor anything else will remove it.

The cache will know nothing of a [binding](../GLOSSARY.md#binding) or an [alias](../GLOSSARY.md#alias). It will store
and return under whatever class it is given. `BindingCatalogue` will expose alias normalisation, and the container will
normalise the class before it consults the cache.

The container will hold two caches:

```php
private InstanceCache $instances        = InstanceCache::strong();
private InstanceCache $liminalInstances = InstanceCache::weak();
```

Reading and writing will both pick a cache by [effective liminality](../GLOSSARY.md#effective-liminality), then delegate
to it. A [liminal](../GLOSSARY.md#liminal) resolution and a shared resolution of the
same class will occupy separate entries.

### Class attributes

The container will memoise, for each class, whether it carries `NoResolution`, `Lazy` and `Liminal`, as
[ADR-0014](../adr/0014-class-level-attributes-are-memoised-as-presence-flags.md) decides. The flags will come from one
unfiltered `getAttributes()` pass, matched on `getName()`, the first time the flags are needed for the class, and no
attribute will be instantiated. Later [resolutions](../GLOSSARY.md#resolution) will read the flags without reflection.

A class-level attribute will apply to a resolution when it is carried by the requested class, after
[alias](../GLOSSARY.md#alias) normalisation, or by the [concrete](../GLOSSARY.md#concrete) class of the resolution's
[binding](../GLOSSARY.md#binding) where it has one. A binding given an object has the object's class as its concrete
class, so that class is consulted. The class of an instance a factory returns is not known before the cache is checked,
and will not be consulted.

Nothing will be memoised about parameters, as
[ADR-0020](../adr/0020-resolution-data-is-not-memoised-per-parameter.md) decides.

### Resolution

These steps will replace the [resolution](../GLOSSARY.md#resolution) steps of
[RFC-0001](0001-dependency-injection-container.md):

1. **Alias.** An [alias](../GLOSSARY.md#alias) is replaced by the [abstract](../GLOSSARY.md#abstract) it points to,
   once. The cache and the [resolution stack](../GLOSSARY.md#resolution-stack) use the class this step produces, and
   class-level attributes are read from it and from the [binding](../GLOSSARY.md#binding)'s
   [concrete](../GLOSSARY.md#concrete) class.
2. **Binding.** The binding for the requested class, name and
   [qualifier](../GLOSSARY.md#qualifier) is looked up. The lookup replaces an alias itself, once, so it reaches the
   same class as the first step.
3. **Liminality.** The resolution is [liminal](../GLOSSARY.md#liminal) if it asks to be, if its binding marks it
   liminal, or if `Liminal` applies to it, as described under Class attributes. This is worked out once, and used for
   both reading and writing the cache.
4. **Cached instance.** If a [shared](../GLOSSARY.md#shared) instance is cached, it is returned: from the weak cache for
   a liminal resolution, while the instance is still alive, and from the strong cache otherwise, under the name or
   qualifier class where the resolution has one.
5. **Lazy proxy.** If the resolution is lazy, its binding marks it lazy, or `Lazy` applies to it, a
   [lazy proxy](../GLOSSARY.md#lazy-proxy) is returned. If `NoResolution` applies and the binding has no instance or
   factory, it throws here instead, as it would at construction. The proxy is made for the binding's concrete class,
   or the class from the first step where there is no concrete class. If that class is an interface, is abstract or
   extends an internal PHP class, it throws `NotInstantiableException`. A class with no properties that could trigger
   initialisation is resolved straight away, as in [RFC-0001](0001-dependency-injection-container.md), so the
   resolution continues to the next step without a proxy. When the proxy is first used, the resolution runs again from
   the first step with this step skipped. If the instance it produces is not one PHP accepts for the proxy, such as an
   instance of a subclass, it throws `DependencyResolutionException`.
6. **Resolution stack.** The resolution's entry is added to the resolution stack, as described under Circular
   dependencies.
7. **Instance.** If the binding holds an instance, that instance is the result. If it has a factory, the factory is
   invoked through the container and its return value is the result.
8. **Construction.** Otherwise the container constructs the binding's concrete class, or the class from the first
   step where there is no concrete class. If `NoResolution` applies to the resolution, it throws. A class that is not
   instantiable throws. A class with no constructor is instantiated directly, and a class with one has its constructor
   invoked through the container.
9. **Sharing.** Unless a binding marks it not shared, the instance is cached: in the weak cache if the resolution is
   liminal, and in the strong cache otherwise, under the class from the first step, and the name or qualifier class
   where the resolution has one.

A resolution will have a name or a qualifier, never both, as a [dependency](../GLOSSARY.md#dependency) cannot in
[RFC-0001](0001-dependency-injection-container.md). `named()` will throw when the resolution already has a qualifier,
and `qualifiedBy()` when it already has a name.

### Circular dependencies

A circular dependency is a [resolution](../GLOSSARY.md#resolution) whose eager construction requires the same resolution
again, directly or through others.

The container will hold a [resolution stack](../GLOSSARY.md#resolution-stack). A resolution will add its entry once it
is past the cached instance and [lazy proxy](../GLOSSARY.md#lazy-proxy) steps, and remove it when it finishes, whether
it returns or throws. A resolution that returns a cached instance or a lazy proxy adds no entry. An entry will be
identified the same way as a [shared](../GLOSSARY.md#shared) instance in the
[instance cache](../GLOSSARY.md#instance-cache): by the class after [alias](../GLOSSARY.md#alias) normalisation, and the
name or [qualifier](../GLOSSARY.md#qualifier) class where the resolution has one.

A resolution whose entry is already on the stack will throw `CircularDependencyException` instead of adding it. Its
message will name every entry in the chain in order, each by its class and by its name or qualifier class where it has
one, from the first appearance of the repeated entry to its repetition, such as `A`, `B`, `C` and then `A` again.

Only eager construction of the same resolution will be detected:

- A class that depends on a [named binding](../GLOSSARY.md#named-binding) or
  [qualified binding](../GLOSSARY.md#qualified-binding) of its own [abstract](../GLOSSARY.md#abstract) is not a
  circular dependency, because the two resolutions have different entries. A decorator bound to `Cache`, whose
  constructor takes the `Cache` [binding](../GLOSSARY.md#binding) named `inner`, resolves normally.
- A resolution already on the stack, requested again lazily, through its binding or through `Lazy`, returns a lazy
  proxy instead of throwing, because a resolution that returns a proxy adds no entry. When the proxy is first used,
  the resolution runs again from the alias step, and a shared resolution whose first construction has finished finds
  its instance in the cache. If the first construction is still on the stack at that point, such as when the proxy is
  used inside the constructor that requested it, that is a circular dependency, and it throws. A class with no
  properties that could trigger initialisation is given no proxy, so a lazy request for one already on the stack is a
  circular dependency.
- A [ghost object](../GLOSSARY.md#ghost-object) is created without a resolution, and its constructor runs when it is
  first accessed, so `Ghost` breaks a circular dependency without touching the stack.
- A class resolved twice in sequence, such as the same class for two parameters of one constructor, is not a circular
  dependency, because the first resolution has left the stack before the second starts.
- A resolution that throws leaves the stack as it was before it started, so an unrelated resolution afterwards is
  not reported as a circular dependency.

### Errors

A circular dependency will throw `CircularDependencyException`, which will be final, extend `RuntimeException` and
implement `ContainerException`. It will be thrown when a [resolution](../GLOSSARY.md#resolution) being constructed
eagerly is required again before it finishes, and will carry the chain of entries in order.

A resolution given both a name and a [qualifier](../GLOSSARY.md#qualifier) will throw `InvalidResolutionException`,
which will be final, extend `InvalidArgumentException` and implement `ContainerException`, as
`InvalidInvocationException` does for an [invocation](../GLOSSARY.md#invocation).

A lazy resolution whose proxy would be made for an interface, an abstract class or a class extending an internal PHP
class will throw `NotInstantiableException`, which [RFC-0001](0001-dependency-injection-container.md) throws for a class
to be constructed that is not instantiable. A lazy proxy whose resolution produces an instance PHP does not accept for
it will throw `DependencyResolutionException` when it is first used, which RFC-0001 throws when a
[dependency](../GLOSSARY.md#dependency) cannot be resolved.

### Out of scope

- **Chained aliases.** An [alias](../GLOSSARY.md#alias) pointing to another alias is not followed: the alias step and
  the [binding](../GLOSSARY.md#binding) lookup each replace an alias once. Whether chains are flattened when a
  [catalogue](../GLOSSARY.md#catalogue) is built, or followed at runtime, belongs to building catalogues.
- **Memoising parameters and resolution plans**, ruled out by
  [ADR-0020](../adr/0020-resolution-data-is-not-memoised-per-parameter.md).

## Alternatives considered

The decisions this design rests on are recorded separately, with the alternatives each one rejected:

- memoising class-level attributes as presence flags, in
  [ADR-0014](../adr/0014-class-level-attributes-are-memoised-as-presence-flags.md)
- not memoising [resolution](../GLOSSARY.md#resolution) data per parameter, in
  [ADR-0020](../adr/0020-resolution-data-is-not-memoised-per-parameter.md)

**One cache holding every instance through weak references.** We rejected this because a weak reference is pointless
beside a strong one in the same store, and a [liminal](../GLOSSARY.md#liminal) and a [shared](../GLOSSARY.md#shared)
resolution of one class are different cache identities: resolving a class without liminality should still cache it
normally.

**Choosing strong or weak references with an enum or a flag.** We rejected this because static factories follow the
container's own convention, such as `Resolution::for()` and `Invocation::callable()`.

**Reordering the existing branches.** Reading and writing would keep their own branches over the four caches, reordered
so that both check them in the same order. We rejected this because it leaves two hand-maintained copies of the same
choice, which liminal named and qualified instances would grow to six caches.

**Matching qualified instances by qualifier value.** A qualified instance would be found by scanning the instances of
its class and comparing [qualifiers](../GLOSSARY.md#qualifier) with `equals()`, as the container does now. We rejected
this because [RFC-0001](0001-dependency-injection-container.md) makes the qualifier's class the key and ignores any
value it carries.

**A return type that depends on how the cache holds its instances.** `get()` would declare a conditional return type
keyed on whether the cache is weak. We rejected this because PHPStan accepts conditional return types only over
parameters and template types, and a call site cannot know which type of cache it holds.

**Reading class-level attributes from one class only.** An attribute would apply only when carried by the requested
class, or only when carried by the class being constructed. We rejected this because a
[binding](../GLOSSARY.md#binding)'s [concrete](../GLOSSARY.md#concrete) class is also an [alias](../GLOSSARY.md#alias)
of its [abstract](../GLOSSARY.md#abstract), so the requested class after normalisation is the abstract whichever of the
two is requested. Reading it alone would never find an attribute on the concrete class, and reading only the class being
constructed would never find one on an interface.

**A name taking precedence over a qualifier.** A resolution given both would resolve as if only the name were given. We
rejected this because the qualifier would be dropped without the caller knowing.

**Resolving eagerly when a proxy cannot be made.** A lazy resolution whose proxy would be made for an interface, an
abstract class or a class extending an internal PHP class would construct its instance at once. We rejected this
because the laziness would be dropped without the caller knowing, and a circular dependency relying on it would be
reported with nothing to say why.

**Adding a stack entry when a resolution starts.** Every resolution would add its entry before the cached instance and
[lazy proxy](../GLOSSARY.md#lazy-proxy) steps. We rejected this because a lazy request for a resolution already on the
[resolution stack](../GLOSSARY.md#resolution-stack) would throw before its proxy could be returned, so laziness would
break a circular dependency only where no lazy request meets its own entry.

**Identifying stack entries by class alone.** We rejected this because a class that depends on a
[named binding](../GLOSSARY.md#named-binding) or [qualified binding](../GLOSSARY.md#qualified-binding) of its own
abstract, such as a decorator, would be reported as a circular dependency.

**Removing a stack entry only when a resolution returns.** We rejected this because a resolution that throws would
leave its entry behind, and an unrelated resolution afterwards would be reported as a circular dependency.

**Removing a collected entry when a read finds it.** `get()` would remove an entry whose instance has been collected.
We rejected this because a read that finds one is followed by construction and a `put()` under the same entry, which
replaces it, unless the resolution returns a lazy proxy that is never used, or throws. Removal on read would free an
entry only in those cases.

**Sweeping collected entries.** The cache would remove every entry whose instance has been collected, on each `put()` or
in a periodic pass. We rejected this because each entry it would free is one empty weak reference, one for each class,
name and qualifier class combination given a liminal resolution. Those combinations grow only when names are built at
runtime, such as one per server, for the life of the worker process that
[ADR-0009](../adr/0009-the-panel-runs-as-a-frankenphp-worker-in-a-single-binary.md) keeps running across requests.
Such names grow the strong cache in the same way, where each entry holds the instance itself, so sweeping the weak cache
would not address that growth.

No other alternatives were weighed.

## Backwards compatibility

These will change for code that resolves through the container today:

- A circular dependency will throw `CircularDependencyException` instead of exhausting the call stack.
- A [shared](../GLOSSARY.md#shared) [binding](../GLOSSARY.md#binding) resolved through an [alias](../GLOSSARY.md#alias)
  more than once will return the same instance. Today each [resolution](../GLOSSARY.md#resolution) through the alias
  constructs a new one, unless the binding holds an instance, and replaces the instance cached under the
  [abstract](../GLOSSARY.md#abstract).
- A resolution through an alias of a binding with no [concrete](../GLOSSARY.md#concrete) class will construct the
  binding's abstract. Today the container constructs the class as requested, so resolving an interface that aliases a
  binding for its implementation throws `NotInstantiableException`.
- A resolution through an alias of an alias will construct and share the class the first alias points to. Today the
  binding lookup finds no binding, and the container constructs the class as requested.
- A shared resolution made [liminal](../GLOSSARY.md#liminal) by its binding or by a class carrying `Liminal` will return
  the same instance while it is alive. Today each resolution that does not itself ask to be liminal constructs a new
  one, unless the binding holds an instance.
- A liminal resolution with a name or [qualifier](../GLOSSARY.md#qualifier) will be cached under it, and a liminal
  resolution without one will no longer return an instance built for a [named binding](../GLOSSARY.md#named-binding)
  or [qualified binding](../GLOSSARY.md#qualified-binding). Today every liminal instance of a class is written under
  the class alone and read under the name or qualifier, so a named or qualified liminal resolution never finds its
  instance.
- Qualified shared instances will be keyed by qualifier class, as [RFC-0001](0001-dependency-injection-container.md)
  specifies, so two qualifiers of one class with different values will share one instance. Today they are matched with
  `Qualifier::equals()` and get separate instances. The container will no longer call `equals()`.
- A class with no binding will be shared, as [RFC-0001](0001-dependency-injection-container.md) specifies. Today it is
  constructed again on every resolution.
- A binding marked lazy will return a [lazy proxy](../GLOSSARY.md#lazy-proxy), as
  [RFC-0001](0001-dependency-injection-container.md) specifies, or throw `NotInstantiableException` where the proxy
  would be made for an interface or an abstract class, such as for a factory binding of an interface. Today the
  container ignores the flag and resolves such a binding at once.
- `Lazy` and `Liminal` will apply when carried by the requested class or by the binding's concrete class, including for
  a binding with an instance or a factory. Today they are read only from the class the container constructs, and only
  when it constructs one. [RFC-0001](0001-dependency-injection-container.md) applies them regardless of any binding,
  but the class of an instance a factory returns will not be consulted.
- `NoResolution` will apply when carried by either class of a resolution the container constructs, so an abstract
  carrying it and bound to a concrete class will throw. Today it is read only from the class the container constructs.
  [RFC-0001](0001-dependency-injection-container.md) applies it only where the binding has no instance, factory or
  concrete class, so this changes its design as well as the code.
- A lazy resolution to which `NoResolution` applies, with no instance or factory, will throw when it is requested.
  Today it returns a lazy proxy, and throws when the proxy is first used.
- A lazy proxy will be made for the binding's concrete class where it has one. Today it is made for the class as
  requested, and a proxy for an interface or an abstract class throws PHP's own `Error`, where it will throw
  `NotInstantiableException`.
- A lazy resolution of a class with no properties that could trigger initialisation will be constructed straight away
  and cached, as [RFC-0001](0001-dependency-injection-container.md) specifies. Today PHP returns an object whose
  constructor never runs, and it is not cached.
- A lazy proxy given an instance PHP does not accept for it, such as a factory's instance of a subclass, will throw
  `DependencyResolutionException` when it is first used. Today PHP throws its own `TypeError`.
- A resolution given both a name and a qualifier will throw `InvalidResolutionException`. Today the name is used and the
  qualifier is ignored.

## Open questions

## Changelog

- 2026-10-05: Reworded in the readable house style, and the project's own vocabulary linked to the glossary. The
  design is unchanged.
- 2026-10-05: Settled collected weak entries. They stay in the weak cache until the next `put()` under the same entry
  replaces them, and removing them on read and sweeping them are recorded as rejected alternatives.
- 2026-10-05: Revised after review. A stack entry is added only once a resolution is past the cached instance and lazy
  proxy steps, so a lazy request breaks a cycle, and `Lazy` is applied before construction. Class-level attributes
  apply when carried by the requested class or the binding's concrete class. The alias step and the binding lookup each
  replace an alias once. A resolution given both a name and a qualifier throws `InvalidResolutionException`. The
  alternatives the issues name are recorded, and Backwards compatibility lists what changes against the code.
- 2026-10-05: Revised after a second review. The lazy proxy step applies the binding's lazy flag, checks
  `NoResolution` first, makes the proxy for the binding's concrete class, throws `NotInstantiableException` where that
  class is an interface or abstract, and runs the resolution again from the alias step. A binding given an object has
  its class consulted for class-level attributes. Alternatives the issues name for the qualifier dimension and the
  stack's key are recorded, the sweeping rejection states its reason, and Backwards compatibility adds the changes the
  revision makes.
- 2026-10-05: Revised after a third review. A class with no properties that could trigger initialisation is resolved
  straight away, as in RFC-0001, instead of being given a lazy proxy. A class extending an internal PHP class throws
  `NotInstantiableException` as interfaces and abstract classes do, and an instance PHP does not accept for a proxy
  throws `DependencyResolutionException`. A circular dependency is no longer called a cycle, and its message names each
  entry's name or qualifier class. Backwards compatibility adds the changes to lazy bindings, `NoResolution`, chained
  aliases and property-less classes, and the Motivation covers aliases, qualifiers and lazy proxies.

## Sources

- Issue [#43], Engine - Container - Memoise class-level attribute lookups, 2026-08-08: memoising class-level
  attributes, and making class-level liminality available when the cache is checked.
- Issue [#44], Engine - Container - Alias cache key mismatch, 2026-08-08, edited the same day to land after [#45]:
  normalising aliases once before the cache is consulted, exposing alias normalisation on `BindingCatalogue`, and
  chained aliases left to building catalogues.
- Issue [#45], Engine - Container - Introduce InstanceCache, 2026-08-08: the instance cache, its strong and weak
  factories, its class and name dimensions, keeping the weak reference internal, two caches rather than weak
  references throughout, static factories rather than an enum or a flag, reordering the branches and a conditional
  return type rejected, alias normalisation staying in the container, and collected weak entries left undecided. It
  kept the qualifier dimension as a linear scan matching qualifiers with `equals()`, which this design does not adopt.
- Issue [#46], Engine - Container - Liminal information parity, 2026-08-08: looking the binding up before the cache
  check, and working out liminality once from the resolution, the binding and the class attribute for both reading
  and writing.
- Issue [#47], Engine - Container - Circular dependency detection, 2026-08-08: the resolution stack, removed however
  the resolution finishes rather than only when it returns, detection observing only eager construction, which it
  proposed to achieve by adding entries on entry, `CircularDependencyException` carrying the chain in order, being final
  and extending `RuntimeException`, `Lazy` on both classes of a circular dependency and `Ghost` on a parameter still
  breaking it, and sequential resolution of one class not counting as a circular dependency. It keyed the stack by class
  alone, which this design does not adopt.
- The instance cache keying qualified instances by qualifier class, resolution stack entries being identified the same
  way as instance cache entries, so that named and qualified child bindings of one abstract are not reported as circular
  dependencies, and the chain running from the repeated entry's first appearance to its repetition: first written down
  on 2026-09-14.
- Collected weak entries being kept until the next `put()` replaces them, with removal on read and sweeping rejected:
  first written down on 2026-10-05.
- Stack entries added only once a resolution is past the cached instance and lazy proxy steps, in place of [#47]'s
  addition on entry, with `Lazy` applied before construction and ghosts not touching the stack; class-level attributes
  applying when carried by the requested class or the binding's concrete class; the alias step and the binding lookup
  each replacing an alias once; a resolution with both a name and a qualifier throwing `InvalidResolutionException`;
  the proxy being made for the binding's concrete class, `NotInstantiableException` when that class is an interface or
  abstract, the proxy's resolution running again from the alias step, and `NoResolution` checked before a proxy is
  returned; and reading attributes from one class only, a name taking precedence over a qualifier, and resolving
  eagerly when a proxy cannot be made, all rejected: first written down on 2026-10-05.
- A class extending an internal PHP class throwing `NotInstantiableException`, an instance PHP does not accept for a
  proxy throwing `DependencyResolutionException`, and the circular dependency message naming each entry's name or
  qualifier class: first written down on 2026-10-05.
- The binding's lazy flag applying at the lazy proxy step, and a class with no properties that could trigger
  initialisation being resolved straight away, as [RFC-0001](0001-dependency-injection-container.md) specifies: first
  written into these steps on 2026-10-05.

[#43]: https://github.com/thegamepanel/panel/issues/43
[#44]: https://github.com/thegamepanel/panel/issues/44
[#45]: https://github.com/thegamepanel/panel/issues/45
[#46]: https://github.com/thegamepanel/panel/issues/46
[#47]: https://github.com/thegamepanel/panel/issues/47
