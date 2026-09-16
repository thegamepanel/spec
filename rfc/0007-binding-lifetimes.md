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

A binding's lifetime replaces its shared flag with three states: `Process`, resolved once and held until the worker
restarts; `Cycle`, resolved once per cycle and discarded when the cycle closes; and `Transient`, a new instance on
every resolution. The container opens and closes cycles for whatever drives it, disposes of cycle instances that hold
resources when a cycle closes, and injects providers where a longer-lived object needs an instance from the current
cycle.

## Motivation

The container records whether a binding is shared, and nothing else about how long an instance lasts. A shared
instance is cached for the life of the container.

The panel runs as a long-lived worker, per
[ADR-0009](../adr/0009-the-panel-runs-as-a-frankenphp-worker-in-a-single-binary.md). When a PHP process lasted one
request, a shared instance lasted one request. In a worker, the same binding lasts until the worker restarts. The
lifetime of every binding changed when the runtime was chosen, without any binding changing.

There is no way to declare one instance per request, discarded afterwards, which anything holding the identity of a
request needs. The same need arises for a job taken by a queue worker, and for a tick of the scheduler.

## Proposal

### Concepts

| Term | Meaning |
|---|---|
| Lifetime | How long a resolved instance is kept: `Process`, `Cycle` or `Transient`. |
| Cycle | A unit of work opened and closed around the container by whatever drives it, such as a request, a job or a tick. |
| Process lifetime | Resolved once, and held until the worker restarts. |
| Cycle lifetime | Resolved once per cycle, and discarded when the cycle closes. |
| Transient lifetime | Resolved anew on every resolution, and never cached. |
| Provider | An object that resolves one class each time it is asked, against the cycle open at that moment. |
| Disposal | Releasing what a cycle instance holds, when its cycle closes. |

### Components

| Component | Responsibility |
|---|---|
| `Lifetime` | Enum of `Process`, `Cycle` and `Transient`. |
| `BindingBuilder`, `Binding` | Carry a binding's lifetime and its disposal callback, in place of the shared flag. |
| `PerProcess`, `PerCycle` | Class attributes declaring the lifetime of a class. |
| `Container` | Opens and closes cycles, caches cycle instances separately, and disposes of them when a cycle closes. |
| `Provider` | Contract for resolving one class on demand. |
| `Provide` | Resolvable attribute naming the class a `Provider` parameter provides. |
| `ProviderResolver` | The resolver paired with `Provide`. |
| `Disposable` | Contract for an instance that releases what it holds when its cycle closes. |
| `LifetimeException` | Thrown when a lifetime or a cycle is used incorrectly. |
| `DisposalException` | Thrown when one or more disposals fail. |

### Lifetimes

| Lifetime | Duration | Cached |
|---|---|---|
| `Process` | Resolved once, held until the worker restarts. | In the strong cache, or the weak cache if liminal. |
| `Cycle` | Resolved once per cycle, discarded when the cycle closes. | In the cycle cache. |
| `Transient` | A new instance on every resolution. | Never. |

A resolution's lifetime is decided in this order:

1. A lifetime declared on its binding.
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

The shared flag is removed, not kept alongside the lifetime.

A class declares its own lifetime with an attribute, which applies whether or not it has a binding, unless its
binding declares one:

```php
#[PerCycle]
final class CurrentUser {}

#[PerProcess]
final class GameCatalogue {}
```

`PerProcess` and `PerCycle` are markers tested only for presence, memoised with the other class attributes, per
[ADR-0014](../adr/0014-class-level-attributes-are-memoised-as-presence-flags.md). There is no attribute for
`Transient`, since a class with no binding is `Transient` already.

### Liminality

Liminality applies to `Process` lifetime alone:

- A liminal `Process` resolution is held in the weak cache.
- A liminal resolution that would otherwise be `Transient` is held weakly as `Process`.
- A liminal resolution of a `Cycle` binding or class stays `Cycle`, and its liminality is ignored.

There is no weak cycle cache. A weakly held instance lasts as long as something else holds it, and a cycle instance
lasts until its cycle closes; combining them gives whichever ends first, and each bound makes the other redundant.

### Cycles

| Method | Effect |
|---|---|
| `openCycle(): void` | Opens a cycle. Throws if one is already open. |
| `closeCycle(): void` | Disposes of the cycle's instances, discards them, and closes the cycle. Throws if none is open. |
| `inCycle(): bool` | Returns whether a cycle is open. |

The container does not know what a cycle represents. Whatever drives it opens and closes cycles: the HTTP worker
around each request, a queue worker around each job, and the scheduler around each tick.

Cycles do not nest. Resolving a `Cycle` lifetime with no cycle open throws, rather than returning an instance with
the wrong identity.

### Caches

The container holds three instance caches, as described in [RFC-0006](0006-container-improvements.md):

```php
private InstanceCache $instances        = InstanceCache::strong();
private InstanceCache $liminalInstances = InstanceCache::weak();
private InstanceCache $cycleInstances   = InstanceCache::strong();
```

One method selects the cache, and both reading and writing call it:

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

In the resolution steps of [RFC-0006](0006-container-improvements.md), the lifetime is decided alongside
liminality, the cached instance step reads from the cache selected for both, and the sharing step writes to it. A
`Transient` resolution reads from and writes to no cache.

Closing a cycle replaces the cycle cache with a new one once its instances are disposed of, so they are discarded as
one object rather than removed one by one. `InstanceCache` itself is unchanged.

### A process instance depending on a cycle instance

A `Process` object whose constructor takes a `Cycle` instance resolves it once, in the first cycle, and holds it for
the life of the worker, so every later cycle sees the first cycle's instance. A lazy proxy or ghost of a `Cycle`
instance has the same result: it resolves once, in whichever cycle first uses it.

The container does not reject this. A longer-lived object that needs an instance from the current cycle takes a
provider instead.

### Providers

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

`Provide` is a resolvable attribute naming the class to provide, and `ProviderResolver` is its resolver, registered
alongside `GhostResolver`, per [RFC-0001](0001-dependency-injection-container.md). The resolver builds a
`Provider` for the class and injects it, so the consuming class never sees the container.

`Provider::get()` resolves the class each time it is called, against whichever cycle is open then, so nothing is
captured. PHP has no generics, so it returns `object`; `Provider<CurrentUser>` exists only as a docblock for static
analysis, and the attribute is the source of truth. A test can substitute its own `Provider` with no container
involved.

`ProviderResolver` checks, when it builds the provider, that the provided class has a binding or can be constructed,
without resolving it, and throws if not. A broken binding therefore fails when the consuming class is built, not on
the first call to `get()`.

Providers are for an object needing an instance with a shorter lifetime than its own. They are not a general way to
defer resolution.

### Disposal

A `Cycle` instance holding a resource, such as an open transaction, a buffered log writer or an unsaved session, is
disposed of when its cycle closes. Disposal is declared in either of two ways:

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

- **`Disposable`**, implemented by the class, suits a class that knows what it holds, with or without a binding.
- **`disposeUsing()`**, declared on the binding, suits a class that cannot or should not implement a container
  contract, such as one from a third-party library. The callback receives the instance. Where a binding declares a
  callback, the callback is used instead of `dispose()`.

`disposeUsing()` is only valid on a `Cycle` binding, and declaring it on any other lifetime throws.

When a cycle closes:

1. The cycle counts as closing, so resolving a `Cycle` lifetime throws.
2. Every instance created during the cycle that is `Disposable`, or whose binding declares a callback, is disposed
   of, in the reverse of the order the instances were created. A dependency is created before whatever depends on
   it, so dependents are disposed of before their dependencies.
3. Every disposal runs, even when one throws.
4. The cycle cache is replaced and the cycle closes.
5. If any disposal threw, a `DisposalException` carrying every failure is thrown.

`Process` instances are not disposed of when a cycle closes.

### Errors

| Exception | Extends | Thrown when |
|---|---|---|
| `LifetimeException` | `LogicException` | A cycle is opened while one is open, closed while none is open, or a `Cycle` lifetime is resolved with no cycle open or while one is closing. `disposeUsing()` is declared on a binding that is not `Cycle`. |
| `DisposalException` | `RuntimeException` | One or more disposals failed when a cycle closed. It carries every failure. |
| `DependencyResolutionException` | `RuntimeException` | The class a `Provider` provides has no binding and cannot be constructed. |

Every exception implements `ContainerException`.

### Out of scope

- **Opening and closing cycles.** The HTTP worker, the queue worker and the scheduler do that, and each belongs to
  its own design.
- **Disposing of `Process` instances** when the worker shuts down, which belongs to bootstrapping.
- **Validating the dependency graph.** A `Process` object taking a `Cycle` instance is not rejected.

## Alternatives considered

The decisions this design rests on are recorded separately:

- running as a long-lived worker, in
  [ADR-0009](../adr/0009-the-panel-runs-as-a-frankenphp-worker-in-a-single-binary.md)
- memoising class-level attributes as presence flags, in
  [ADR-0014](../adr/0014-class-level-attributes-are-memoised-as-presence-flags.md)

**Naming the middle lifetime `Request`.** HTTP opens a cycle for each request, but a queue worker opens one for each
job and the scheduler for each tick. `Request` would put HTTP vocabulary into the container, and read wrongly for the
other two.

**Keeping the shared flag alongside the lifetime.** Two ways of saying the same thing would disagree.

**Resolving a `Cycle` lifetime outside a cycle as `Transient`.** It would produce an instance with the wrong identity,
and no error.

**Nesting cycles.** A second cycle opened inside the first would need a stack of cycle caches, and nothing needs one.

**Partitioning `InstanceCache` by lifetime.** A third cache keeps `InstanceCache` unchanged, and closing a cycle
discards one object instead of walking entries.

**A forwarding proxy in place of a provider**, built on the ghost machinery. It would resolve a different instance
between two calls on the same object, with nothing visible where it is used.

No other alternatives were weighed.

## Backwards compatibility

- `notShared()` is replaced by `transient()`, and a binding's shared flag by its lifetime.
- A class with no binding is `Transient`, where [RFC-0001](0001-dependency-injection-container.md) shares it.

## Open questions

## Changelog

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
