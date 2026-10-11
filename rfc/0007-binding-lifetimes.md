---
id: RFC-0007
title: Binding lifetimes
status: accepted
created: 2026-09-15
decided: 2026-10-11
depends: [ADR-0009, ADR-0014]
updates: [RFC-0001, RFC-0006]
obsoletes: []
---

# RFC-0007: Binding lifetimes

## Abstract

We will replace the shared flag on a [binding](../GLOSSARY.md#binding) with a [lifetime](../GLOSSARY.md#lifetime) of
three states: `Process`, resolved once and held until the worker restarts; `Cycle`, resolved once per
[cycle](../GLOSSARY.md#cycle) and discarded when the cycle closes; and `Transient`, a new instance on every resolution.
The container will open and close cycles for whatever drives it, and dispose of cycle instances that hold resources when
a cycle closes. Where a longer-lived object needs an instance from the current cycle, the container will inject a
[provider](../GLOSSARY.md#provider).

## Motivation

The container records whether a [binding](../GLOSSARY.md#binding) is [shared](../GLOSSARY.md#shared), and nothing else
about how long an instance lasts. A shared instance is cached for the life of the container.

The panel runs as a long-lived worker, as
[ADR-0009](../adr/0009-the-panel-runs-as-a-frankenphp-worker-in-a-single-binary.md) decides. When a PHP process lasted
one request, a shared instance lasted one request. In a worker, the same binding lasts until the worker restarts. The
[lifetime](../GLOSSARY.md#lifetime) of every binding changed when the runtime was chosen, without any binding changing.

There is no way to declare one instance per request, discarded afterwards, which anything holding the identity of a
request needs. The same need arises for a job taken by a queue worker, and for a tick of the scheduler.

## Proposal

### Lifetimes

Every [resolution](../GLOSSARY.md#resolution) will have a [lifetime](../GLOSSARY.md#lifetime), one of the three cases of
the `Lifetime` enum:

| Lifetime | Duration | Cached |
|---|---|---|
| `Process` | Resolved once, held until the worker restarts. | In the strong cache, or the weak cache if liminal. |
| `Cycle` | Resolved once per cycle, discarded when the cycle closes. | In the cycle cache. |
| `Transient` | A new instance on every resolution. | Never. |

A resolution's lifetime will be decided in this order:

1. A lifetime declared on its [binding](../GLOSSARY.md#binding). A binding holding an instance declares `Process`.
2. Otherwise, a lifetime declared with `PerProcess` or `PerCycle` by the requested class, after
   [alias](../GLOSSARY.md#alias) normalisation, or by the binding's [concrete](../GLOSSARY.md#concrete) class, the two
   classes [RFC-0006](0006-container-improvements.md) reads class-level attributes from. Where the two classes declare
   different lifetimes, or one class carries both attributes, the resolution throws `LifetimeException`.
3. Otherwise, `Process` for a class with a binding, and `Transient` for a class without one.
4. A [liminal](../GLOSSARY.md#liminal) resolution whose lifetime would be `Transient` is `Process`, as described under
   Liminality.

### Declaring a lifetime

```php
$registry->bind(ServerRepository::class)->to(SqlServerRepository::class);                  // Process
$registry->bind(CurrentUser::class)->using($factory)->lifetime(Lifetime::Cycle);           // Cycle
$registry->bind(Report::class)->transient();                                               // Transient
```

| Builder method | Effect |
|---|---|
| `lifetime(Lifetime $lifetime)` | Declares the binding's lifetime. |
| `transient()` | Declares the binding `Transient`. It replaces `notShared()`. |
| `disposeUsing(Closure $callback)` | Declares how an instance of a `Cycle` binding is disposed, as described under Disposal. |

`BindingBuilder` and `Binding` will carry the [lifetime](../GLOSSARY.md#lifetime) of a [binding](../GLOSSARY.md#binding)
and its disposal callback in place of the shared flag. The shared flag will be removed, not kept alongside the lifetime.

A binding holding an instance will always be `Process`, since its instance is the result of every resolution. Declaring
`Cycle` or `Transient` on one will throw `LifetimeException` when the binding is built from its builder, whatever order
the builder's methods were called in.

A class will declare its own lifetime with an attribute, which will apply whether or not it has a binding, unless its
binding declares one:

```php
#[PerCycle]
final class CurrentUser {}

#[PerProcess]
final class GameCatalogue {}
```

An abstract and its concrete class that both declare a lifetime will have to declare the same one. A binding's declared
lifetime overrides the attributes of both classes, so the binding is where an implementation is given a lifetime
different from the one its abstract declares:

```php
$registry->bind(PermissionChecker::class)->to(CachingPermissionChecker::class)->lifetime(Lifetime::Cycle);
```

`PerProcess` and `PerCycle` will be markers tested only for presence, memoised with the other class attributes, as
[ADR-0014](../adr/0014-class-level-attributes-are-memoised-as-presence-flags.md) decides. There will be no attribute for
`Transient`, since a class with no binding will be `Transient` already.

### Liminality

Liminality will apply to the `Process` [lifetime](../GLOSSARY.md#lifetime) alone:

- A [liminal](../GLOSSARY.md#liminal) `Process` resolution will be held in the weak cache.
- A liminal resolution that would otherwise be `Transient` will be held weakly as `Process`.
- A liminal resolution of a `Cycle` binding or class will stay `Cycle`, and its liminality will be ignored.

There will be no weak cycle cache. A weakly held instance lasts as long as something else holds it, and a
[cycle](../GLOSSARY.md#cycle) instance will last until its cycle closes. Combining them would give whichever ends first,
and each bound would make the other redundant.

### Cycles

| Method | Effect |
|---|---|
| `openCycle(): void` | Opens a cycle. Throws if one is already open. |
| `closeCycle(): void` | Disposes of the cycle's instances, discards them, and closes the cycle. Throws if none is open. |
| `inCycle(): bool` | Returns whether a cycle is open. |

The container will not know what a [cycle](../GLOSSARY.md#cycle) represents. Whatever drives it will open and close
cycles, such as a worker serving HTTP around each request, a queue worker around each job, or a scheduler around each
tick.

Cycles will not nest. Resolving a `Cycle` [lifetime](../GLOSSARY.md#lifetime) with no cycle open will throw, rather than
return an instance with the wrong identity. The check will be made when the lifetime is decided, so a lazy resolution of
a `Cycle` lifetime will throw when it is requested with no cycle open, and its
[lazy proxy](../GLOSSARY.md#lazy-proxy) will throw when it is first used with none open.

### Caches

The container will hold three caches of the [instance cache](../GLOSSARY.md#instance-cache) type described in
[RFC-0006](0006-container-improvements.md):

```php
private InstanceCache $instances        = InstanceCache::strong();
private InstanceCache $liminalInstances = InstanceCache::weak();
private InstanceCache $cycleInstances   = InstanceCache::strong();
```

One method will select the cache, and both reading and writing will call it:

```php
private function cacheFor(bool $liminal, Lifetime $lifetime): ?InstanceCache
{
    return match (true) {
        $lifetime === Lifetime::Transient => null,
        $lifetime === Lifetime::Cycle     => $this->cycleInstances,
        $liminal                          => $this->liminalInstances,
        default                           => $this->instances,
    };
}
```

In the [resolution](../GLOSSARY.md#resolution) steps of [RFC-0006](0006-container-improvements.md), the
[lifetime](../GLOSSARY.md#lifetime) will be decided alongside liminality. The cached instance step will read from the
cache selected for both, and the sharing step will write to it. A `Transient` resolution will read from and write to no
cache. The lifetime passed to `cacheFor()` will be the one decided after liminality is applied, so a liminal resolution
never reaches it as `Transient`.

Closing a [cycle](../GLOSSARY.md#cycle) will replace the cycle cache with a new one once its instances are disposed of,
so they will be discarded as one object rather than removed one by one. `InstanceCache` itself will not change.

### Ghost objects

A [dependency](../GLOSSARY.md#dependency) resolved as a [ghost object](../GLOSSARY.md#ghost-object) will follow its
[lifetime](../GLOSSARY.md#lifetime). A ghost becomes the instance itself once initialised, as
[RFC-0001](0001-dependency-injection-container.md) describes, so it can be cached as the instance before it is
initialised. `GhostResolver` will decide the lifetime as for a [resolution](../GLOSSARY.md#resolution) of the same class
and name or [qualifier](../GLOSSARY.md#qualifier), and use the cache selected for it:

- An instance already in that cache will be injected as it is, with no ghost created.
- Otherwise the ghost will be created and put in that cache at once, so a later resolution of the same class returns
  the same object: for the rest of the [cycle](../GLOSSARY.md#cycle) for a `Cycle` lifetime, and until the worker
  restarts for `Process`.
- A `Transient` lifetime will create a new ghost each time, cached nowhere.

A ghost will be added to the [disposal list](../GLOSSARY.md#disposal-list) when it is initialised, after the
dependencies its constructor resolves, rather than when it is created. A ghost never initialised will not be disposed
of.

### A process instance depending on a cycle instance

A `Process` object whose constructor takes a `Cycle` instance will resolve it once, in the first
[cycle](../GLOSSARY.md#cycle), and hold it for the life of the worker, so every later cycle will see the first cycle's
instance. A [lazy proxy](../GLOSSARY.md#lazy-proxy) of a `Cycle` instance will have the same result: it will resolve
once, in whichever cycle first uses it. So will a [ghost object](../GLOSSARY.md#ghost-object) held by a `Process`
object, which is initialised once, in whichever cycle first uses it.

The container will not reject this. A longer-lived object that needs an instance from the current cycle will take a
[provider](../GLOSSARY.md#provider) instead.

### Providers

A [provider](../GLOSSARY.md#provider) will implement the `Provider` contract:

```php
interface Provider
{
    public function get(): object;
}
```

```php
final class AuditLogger
{
    public function __construct(
        #[Provide(CurrentUser::class)] private Provider $currentUser,
    ) {}

    public function log(string $event): void
    {
        $user = $this->currentUser->get();
    }
}
```

`Provide` will be a [resolvable attribute](../GLOSSARY.md#resolvable-attribute) naming the class to provide, and
`ProviderResolver` its [resolver](../GLOSSARY.md#resolver), registered alongside `GhostResolver`, as
[RFC-0001](0001-dependency-injection-container.md) describes. The resolver will build a `Provider` for the class and
inject it, so the consuming class never sees the container.

`Provider::get()` will resolve the class each time it is called, against whichever [cycle](../GLOSSARY.md#cycle) is open
then, so nothing will be captured. PHP has no generics, so it will return `object`; `Provider<CurrentUser>` will exist
only as a docblock for static analysis, and the attribute will be the source of truth. A test will be able to substitute
its own `Provider` with no container involved.

`ProviderResolver` will check, when it builds the provider, that the provided class has a
[binding](../GLOSSARY.md#binding) or can be constructed, without resolving it, and will throw if not. A broken binding
will therefore fail when the consuming class is built, not on the first call to `get()`.

Providers will be for an object needing an instance with a shorter [lifetime](../GLOSSARY.md#lifetime) than its own.
They will not be a general way to defer resolution.

### Disposal

A `Cycle` instance holding a resource, such as an open transaction, a buffered log writer or an unsaved session, will be
disposed of when its [cycle](../GLOSSARY.md#cycle) closes. [Disposal](../GLOSSARY.md#disposal) will be declared in
either of two ways:

```php
interface Disposable
{
    public function dispose(): void;
}
```

```php
$registry->bind(Session::class)
    ->lifetime(Lifetime::Cycle)
    ->disposeUsing(fn (Session $session) => $session->save());
```

- **`Disposable`**, implemented by the class, will suit a class that knows what it holds, with or without a
  [binding](../GLOSSARY.md#binding).
- **`disposeUsing()`**, declared on the binding, will suit a class that cannot or should not implement a container
  contract, such as one from a third-party library. The callback will receive the instance. Where a binding declares a
  callback, the callback will be used instead of `dispose()`.

`disposeUsing()` will only be valid on a binding that declares the `Cycle` [lifetime](../GLOSSARY.md#lifetime) itself.
Declaring it on a binding that does not will throw `LifetimeException` when the binding is built from its builder, even
where a `PerCycle` attribute would make its resolutions `Cycle`. A class that disposes of itself, whatever declares its
lifetime, will implement `Disposable`.

Only `Cycle` instances will be disposed of. `Process` and `Transient` instances will not be, even when they are
`Disposable` or created during a cycle.

The container will keep a [disposal list](../GLOSSARY.md#disposal-list) for the open cycle, separate from the cycle
cache. A `Cycle` instance that is `Disposable`, or whose binding declares a callback, will be appended to it once
constructed, and every other instance will be left off it. A [ghost object](../GLOSSARY.md#ghost-object) is constructed
when it is initialised. A dependency resolved eagerly is constructed before whatever depends on it, so it will appear
earlier in the list. A dependency constructed later, through a [lazy proxy](../GLOSSARY.md#lazy-proxy) or a ghost
object, will appear after whatever holds it.

When a cycle closes:

1. The cycle counts as closing, so resolving a `Cycle` lifetime throws.
2. Every instance on the disposal list is disposed of, from the last appended to the first, so an instance is disposed
   of before the dependencies it resolved eagerly, and after any constructed later through a lazy proxy or a ghost.
3. Every disposal runs, even when one throws.
4. The cycle cache is replaced, the disposal list is dropped, and the cycle closes.
5. If any disposal threw, a `DisposalException` carrying every failure is thrown.

### Errors

| Exception | Extends | Thrown when |
|---|---|---|
| `LifetimeException` | `LogicException` | A cycle is opened while one is open, closed while none is open, or a `Cycle` lifetime is resolved with no cycle open or while one is closing. A resolution's two classes declare different lifetimes, or one class carries both `PerProcess` and `PerCycle`. A binding holding an instance declares `Cycle` or `Transient`, or `disposeUsing()` is declared on a binding that does not declare `Cycle`. |
| `DisposalException` | `RuntimeException` | One or more disposals failed when a cycle closed. It carries every failure. |
| `DependencyResolutionException` | `RuntimeException` | The class a `Provider` provides has no binding and cannot be constructed. |

Every exception will implement `ContainerException`.

### Out of scope

- **Opening and closing cycles.** The HTTP worker, the queue worker and the scheduler do that, and each belongs to its
  own design.
- **Disposing of `Process` instances** when the worker shuts down, which belongs to bootstrapping.
- **Validating the dependency graph.** A `Process` object taking a `Cycle` instance is not rejected.

## Alternatives considered

The decisions this design rests on are recorded separately:

- running as a long-lived worker, in [ADR-0009](../adr/0009-the-panel-runs-as-a-frankenphp-worker-in-a-single-binary.md)
- memoising class-level attributes as presence flags, in
  [ADR-0014](../adr/0014-class-level-attributes-are-memoised-as-presence-flags.md)

**Naming the middle lifetime `Request`.** We rejected this because HTTP opens a [cycle](../GLOSSARY.md#cycle) for each
request, but a queue worker opens one for each job and the scheduler for each tick. `Request` would put HTTP
vocabulary into the container, and read wrongly for the other two.

**Keeping the shared flag alongside the lifetime.** We rejected this because two ways of saying the same thing would
disagree.

**Resolving a `Cycle` lifetime outside a cycle as `Transient`.** We rejected this because it would produce an instance
with the wrong identity, and no error.

**Nesting cycles.** We rejected this because a second cycle opened inside the first would need a stack of cycle
caches, and nothing needs one.

**Partitioning `InstanceCache` by lifetime.** We rejected this because a third cache keeps `InstanceCache` unchanged,
and closing a cycle discards one object instead of walking entries.

**A forwarding proxy, built on the ghost object machinery, in place of a provider.** We rejected this because it would
resolve a different instance between two calls on the same object, with nothing visible where it is used.

**Sharing a class with no binding**, as [RFC-0001](0001-dependency-injection-container.md) and
[RFC-0006](0006-container-improvements.md) specify. Every class the container constructs without a
[binding](../GLOSSARY.md#binding) would be `Process`. We rejected this because in a worker, a class that keeps state
and that nobody bound, such as one accumulating invoice lines, would carry that state into every later request and grow
until the worker restarts. It would be easy to do, and not every class can be expected to be bound.

**Letting an implementation shorten its abstract's lifetime.** A concrete class could declare a shorter
[lifetime](../GLOSSARY.md#lifetime) than its abstract, as a return type may narrow, but not a longer one. We rejected
this because a lifetime promises both sharing, which a longer one gives more of, and isolation, which a shorter one
gives more of, so neither direction keeps every promise. A `Process` abstract may be injected directly into a `Process`
object, and a `Cycle` implementation of it would be captured there from the first [cycle](../GLOSSARY.md#cycle), with
nothing visible where it is used.

**The concrete class's lifetime winning.** The [concrete](../GLOSSARY.md#concrete) class's attribute would apply, and
the abstract's only where there is no concrete class. We rejected this because whatever depends on the abstract could
not rely on the lifetime it declares.

**Leaving ghost objects outside lifetimes.** A [ghost object](../GLOSSARY.md#ghost-object) would stay its own instance,
created for each dependency and never cached, as in [RFC-0001](0001-dependency-injection-container.md). We rejected
this because a ghost of a `Cycle` or `Process` lifetime would be a second instance beside the cached one, and a second
`Cycle` instance would never be disposed of, so writes to it, such as to a session, would be lost.

**Ignoring the lifetime of a binding holding an instance.** A `Cycle` or `Transient` lifetime declared on one would be
accepted and have no effect. We rejected this because a `Cycle` or `Transient` lifetime
cannot be honoured when every resolution returns the same instance, so declaring one is a mistake to report.

**Accepting `disposeUsing()` wherever the resolved lifetime is `Cycle`.** The check would be made at resolution, after
the class attributes are read. We rejected this because a disposal callback on a binding that does not declare `Cycle`
would be attached for nothing, and the mistake would surface only when the binding is first resolved. A class that
disposes of itself implements `Disposable`.

**Making `disposeUsing()` declare the `Cycle` lifetime.** Declaring a disposal callback on a binding would make it
`Cycle`. We rejected this because a lifetime set as a side effect of
another method is easy to miss when reading the binding.

No other alternatives were weighed.

## Backwards compatibility

- `notShared()` will be replaced by `transient()`, and the shared flag on a [binding](../GLOSSARY.md#binding), and the
  `shared` argument of `Binding`'s constructor, by its [lifetime](../GLOSSARY.md#lifetime).
- A class with no binding will be `Transient`, where [RFC-0001](0001-dependency-injection-container.md) and
  [RFC-0006](0006-container-improvements.md) share it.
- A binding holding an instance and declared `transient()` will throw. Today `notShared()` on one has no effect, and
  every resolution returns its instance.
- A [ghost object](../GLOSSARY.md#ghost-object) of a `Process` lifetime will be the shared instance, or be cached as
  it. Today every ghost is a separate instance, never cached.

## Open questions

## Changelog

- 2026-10-05: Reworded in the readable house style, and the project's own vocabulary linked to the glossary. The
  design is unchanged.
- 2026-10-11: Revised after review, against [RFC-0006](0006-container-improvements.md) as accepted. A lifetime attribute
  is read from the requested class and the binding's concrete class, and a disagreement between them, or one class
  carrying both attributes, throws. A binding holding an instance is always `Process`. `disposeUsing()` requires the
  binding to declare `Cycle` itself. Only `Cycle` instances are disposed of, in reverse order from a disposal list kept
  beside the cycle cache, so a dependency constructed later through a lazy proxy or a ghost is disposed of before
  whatever holds it. Ghost objects follow their lifetime. A liminal resolution that would be `Transient` is
  `Process` before the cache is chosen, and a lazy `Cycle` resolution with no cycle open throws when requested.
  [RFC-0006](0006-container-improvements.md) moves from `depends` to `updates`. The alternatives for each, and the
  reason a class with no binding is `Transient`, are recorded.

## Sources

- Issue [#63], Engine - Container - Binding lifetimes, 2026-08-21, edited on 2026-08-22: the motivation, the three
  lifetimes and why the middle one is not named `Request`, the defaults, `lifetime()`, `transient()` and the removal
  of the shared flag, `PerCycle`, the third cache and its single selector, liminality with cycles not built, opening
  and closing cycles, throwing outside a cycle and for nested cycles, a process instance depending on a cycle
  instance, providers, disposal in reverse order with its contract left open, and the boundaries. Its edit of
  2026-08-22 added `Provide`, `ProviderResolver`, the check when the provider is built, and providers being for a
  lifetime mismatch only. A comment of the same day records why: the first consumer, in sessions, showed that a
  parameter typed `Provider` carries nothing saying what to provide.
- A class with no binding being `Transient`, with no attribute for it; the `PerProcess` attribute; the order a
  lifetime is decided in; liminality applying to `Process` alone, with a liminal resolution of a `Cycle` lifetime
  staying `Cycle`; the provider check meaning a binding or a constructible class; lazy proxies and ghosts of cycle
  instances resolving once; disposal through both `Disposable` and `disposeUsing()`, with the binding's callback
  used instead of `dispose()` and valid only on `Cycle` bindings; the rules for a closing cycle; the cycle methods;
  and the exceptions: first written down on 2026-09-15.
- The disposal list and its order; reading a lifetime attribute from both classes and throwing
  when they disagree; a binding holding an instance being `Process`; `disposeUsing()` requiring a declared `Cycle`;
  disposal of `Cycle` instances only; ghost objects following their lifetime; the reason a class with no binding is
  `Transient`; and the alternatives rejected for each: first written down on 2026-10-11.

[#63]: https://github.com/thegamepanel/panel/issues/63
