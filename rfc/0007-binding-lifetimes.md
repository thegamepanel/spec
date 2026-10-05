---
id: RFC-0007
title: Binding lifetimes
status: proposed
created: 2026-09-15
decided:
depends: [ADR-0009, ADR-0014, RFC-0006]
updates: [RFC-0001]
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

1. A lifetime declared on its [binding](../GLOSSARY.md#binding).
2. A lifetime declared by its class, with `PerProcess` or `PerCycle`.
3. Otherwise, `Process` for a class with a binding, and `Transient` for a class without one.

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

A class will declare its own lifetime with an attribute, which will apply whether or not it has a binding, unless its
binding declares one:

```php
#[PerCycle]
final class CurrentUser {}

#[PerProcess]
final class GameCatalogue {}
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
return an instance with the wrong identity.

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
cache.

Closing a [cycle](../GLOSSARY.md#cycle) will replace the cycle cache with a new one once its instances are disposed of,
so they will be discarded as one object rather than removed one by one. `InstanceCache` itself will not change.

### A process instance depending on a cycle instance

A `Process` object whose constructor takes a `Cycle` instance will resolve it once, in the first
[cycle](../GLOSSARY.md#cycle), and hold it for the life of the worker, so every later cycle will see the first cycle's
instance. A [lazy proxy](../GLOSSARY.md#lazy-proxy) or [ghost object](../GLOSSARY.md#ghost-object) of a `Cycle` instance
will have the same result: it will resolve once, in whichever cycle first uses it.

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

`disposeUsing()` will only be valid on a `Cycle` binding, and declaring it on any other
[lifetime](../GLOSSARY.md#lifetime) will throw.

When a cycle closes:

1. The cycle counts as closing, so resolving a `Cycle` lifetime throws.
2. Every instance created during the cycle that is `Disposable`, or whose binding declares a callback, is disposed
   of, in the reverse of the order the instances were created. A dependency is created before whatever depends on
   it, so dependents are disposed of before their dependencies.
3. Every disposal runs, even when one throws.
4. The cycle cache is replaced and the cycle closes.
5. If any disposal threw, a `DisposalException` carrying every failure is thrown.

`Process` instances will not be disposed of when a cycle closes.

### Errors

| Exception | Extends | Thrown when |
|---|---|---|
| `LifetimeException` | `LogicException` | A cycle is opened while one is open, closed while none is open, or a `Cycle` lifetime is resolved with no cycle open or while one is closing. `disposeUsing()` is declared on a binding that is not `Cycle`. |
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

No other alternatives were weighed.

## Backwards compatibility

- `notShared()` will be replaced by `transient()`, and the shared flag on a [binding](../GLOSSARY.md#binding) by its
  [lifetime](../GLOSSARY.md#lifetime).
- A class with no binding will be `Transient`, where [RFC-0001](0001-dependency-injection-container.md) shares it.

## Open questions

## Changelog

- 2026-10-05: Reworded in the readable house style, and the project's own vocabulary linked to the glossary. The
  design is unchanged.

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

[#63]: https://github.com/thegamepanel/panel/issues/63
