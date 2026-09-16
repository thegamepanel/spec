---
id: RFC-0009
title: Events
status: proposed
created: 2026-09-16
decided:
depends: [ADR-0003, RFC-0001, RFC-0007]
updates: []
obsoletes: []
---

# RFC-0009: Events

## Abstract

A synchronous, in-process event dispatcher. Any object is an event, and listeners are registered against an event
type as references, sealed into an immutable catalogue. A listener matches an event it is typed against, or any
subtype of it, and the matched set for a concrete event class is worked out on its first dispatch and kept. An event
may opt into cancellation, which halts propagation.

## Motivation

Something that happens in one part of the panel often matters to another. A server being deleted matters to whatever
cleans up after it, and to whatever writes an audit trail, neither of which the code deleting the server should have
to know about.

Modules make that sharper. A module needs to react to what the engine and other modules do, without either side
depending on the other.

## Proposal

### Concepts

| Term | Meaning |
|---|---|
| Event | Any object, dispatched so that whatever is interested can act on it. |
| Listener | One handler for one event type. The only thing the dispatcher stores and calls. |
| Subscriber | A class declaring several listeners through attributes on its methods. |
| Handler reference | How a listener is stored: an invokable class, or a class and a method name. |
| Match set | The listeners that match one concrete event class, in registration order. |

### Components

| Component | Responsibility |
|---|---|
| `EventRegistry` | Mutable. Collects registrations of an event type against a handler reference. |
| `EventCatalogue` | Immutable. The registrations the dispatcher reads, sealed from the registry. |
| `EventDispatcher` | Dispatches an event to every matching listener. |
| `Cancellable` | Contract an event implements to be cancellable. |

### Events

Any object is an event. There is no marker interface to implement and nothing to extend: the class is the identity.

Naming carries the meaning. The past tense says it has happened and cannot be stopped, such as `ServerDeleted`. The
present participle says it is about to happen and may be cancellable, such as `ServerDeleting`.

An event that wants something back from its listeners carries whatever collects it, such as a registry or a builder,
and listeners call into that. The dispatcher gathers nothing and aggregates nothing.

### Registration

Registration follows the pattern in
[ADR-0003](../adr/0003-mutable-registries-are-sealed-into-immutable-catalogues.md): `EventRegistry` collects, and
`seal()` produces the `EventCatalogue` the dispatcher reads. Registering after the registry has been sealed throws.

```php
$registry->listen(ServerDeleted::class, PurgeServerFiles::class);
$registry->listen(ServerDeleting::class, [BlockDeletionWhileRunning::class, 'handle']);

$catalogue = $registry->seal();
```

| Method | Effect |
|---|---|
| `listen(string $event, string\|array $handler)` | Registers an invokable class, or a class and method, as a listener for the event type. |
| `seal(): EventCatalogue` | Produces the immutable catalogue and closes registration. |

A registration holds a reference, never an instance. A listener for an event that never fires is never constructed.

A subscriber, a class declaring several listeners through attributes on its methods, is expanded into registrations
by whatever reads those attributes. That is reflection over attributes at registration time, which belongs to the
module system, and the dispatcher has no concept of a subscriber.

### Dispatch

```php
public function dispatch(object $event): object;
```

The event is passed to each matching listener in turn, in the order the listeners were registered, and then returned.
The dispatcher returns the event itself, so anything a caller wants back it reads from the event.

A listener is resolved through the container, per [RFC-0001](0001-dependency-injection-container.md), the first time
an event it matches is dispatched, and is `Process` lifetime, per [RFC-0007](0007-binding-lifetimes.md). A listener
holds no state belonging to a request, and reaches anything that does through a provider.

### Matching

A listener matches an event that is the type it is registered against, extends it, or implements it. A listener
registered against `object` matches every event.

A marker interface on a set of events, with one listener registered against that interface, therefore reaches every
event carrying it, including events defined later by a module.

Matching cannot be worked out in advance. Any object is an event, so the set of concrete event classes is open and
unknown until one is dispatched. The match set for a concrete class is worked out on its first dispatch and kept for
the life of the worker, so later dispatches of that class read it directly.

### Cancellation

An event opts into cancellation by implementing `Cancellable`:

```php
interface Cancellable
{
    public function cancel(string $reason): void;

    public function isCancelled(): bool;

    public function reason(): ?string;
}
```

Cancelling halts propagation: no listener after the one that cancelled runs. `dispatch()` returns the event, and the
caller reads `isCancelled()` and `reason()` to find out what happened and why.

Because propagation stops at the first cancellation, the first reason is the one the event carries.

### Listeners that throw

An exception from a listener propagates. The dispatcher catches nothing, and no listener after the one that threw
runs.

What that means for the caller is the caller's own concern. Work inside a transaction is rolled back by that
transaction. Whatever dispatches an event around a boundary it must close, such as a driver closing a cycle, does so
inside its own `try`/`finally`, so a throwing listener cannot skip the teardown.

### Errors

| Exception | Thrown when |
|---|---|
| `EventLifecycleException` | Registration is attempted after the registry has been sealed. |

A handler reference that cannot be resolved fails when the listener is first needed, with the exception the container
raises, per [RFC-0001](0001-dependency-injection-container.md).

### Out of scope

- **Which events exist.** What is dispatched, and what an event carries, is decided where it is dispatched.
- **Subscribers and attribute extraction.** Reading listener attributes off a class belongs to the module system.
- **Listener priority.** Listeners run in registration order, and nothing reorders them.
- **Durable delivery.** Delivering an event beyond the process, with an outbox for durability, is a separate
  component built on this one.
- **An audit trail.** Separate again, and most likely a module.
- **Collecting contributions from modules.** A collector asks every module for contributions when a component
  chooses; an event fires at a moment in a flow. They are different mechanisms.
- **Dispatching from the container.** Whatever owns a boundary dispatches around it. The container never dispatches.

## Alternatives considered

**Working out every match when the catalogue is sealed.** The set of concrete event classes is open, because any
object is an event, so there is nothing complete to precompute. Matching per concrete class on first dispatch, and
keeping the result, gives the same saving without needing the set in advance.

No other alternatives were weighed.

## Backwards compatibility

Nothing breaks. No event dispatcher exists before this.

## Open questions

## Changelog

## Sources

- Issue [#68], Engine - Events, 2026-08-22, with no edits and no comments: a synchronous in-process dispatcher, any
  object being an event with naming carrying the meaning, events carrying whatever gathers contributions, listeners
  and subscribers, registration of handler references through a registry sealed into a catalogue, references rather
  than instances, `dispatch(object $event): object`, matching over class, parent and interface with `object` matching
  everything, matching resolved on first dispatch of a concrete class and kept for the life of the worker,
  `Cancellable` halting propagation, listeners resolved from the container as `Process` lifetime and reaching
  cycle-scoped state through a provider, a throwing listener propagating, and every boundary listed under Out of
  scope.
- Issue [#65], Engine - HTTP - Middleware pipeline, 2026-08-21: the shape the memoised match set follows, a set
  composed on first use and kept.
- Issues [#51] and [#61], Engine - Database - PostgreSQL and its primitives, 2026-08-19: the notify primitives and
  the outbox that durable delivery will be built on, which is why it is a separate component.
- The name `EventDispatcher`, the `listen()` and `seal()` registry methods, `reason()` on `Cancellable` with the
  first reason being the one carried, `EventLifecycleException` for registration after sealing, and listeners having
  no priority: first written down on 2026-09-16.

[#51]: https://github.com/thegamepanel/panel/issues/51
[#61]: https://github.com/thegamepanel/panel/issues/61
[#65]: https://github.com/thegamepanel/panel/issues/65
[#68]: https://github.com/thegamepanel/panel/issues/68
