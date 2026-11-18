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

I will add a synchronous, in-process event dispatcher in which any object can be an event. Listeners will be registered
against an event type by [handler reference](../GLOSSARY.md#handler-reference) and sealed into an immutable
[catalogue](../GLOSSARY.md#catalogue). A listener will receive every event of its type or of a subtype, and an event
will be able to opt into cancellation, which stops every later listener from running.

## Motivation

Something that happens in one part of the panel often matters to another. When a server is deleted, whatever cleans up
after it needs to know, and so does whatever keeps an audit trail, but the code deleting the server should not have to
know about either.

This matters more for a [module](../GLOSSARY.md#module), which has to react to what the [engine](../GLOSSARY.md#engine)
and other modules do without either side depending on the other.

## Proposal

### What an event is

Any object will be an event. There will be no marker interface to implement and no base class to extend: the event's
class will be its identity.

The class name will carry the meaning. A name in the past tense, such as `ServerDeleted`, will say the thing has
happened and cannot be stopped. A name in the present participle, such as `ServerDeleting`, will say it is about to
happen, and such an event may be cancellable.

An event that needs something back from its listeners will carry an object that collects it, such as a
[registry](../GLOSSARY.md#registry) or a builder, and its listeners will call into that. The dispatcher will gather and
combine nothing itself.

### Listeners

A listener will be one handler for one event type, and it will be the only thing the dispatcher stores and calls. It
will be registered as a [handler reference](../GLOSSARY.md#handler-reference), never as an instance, so a listener for
an event that never fires will never be constructed.

Registration will follow [ADR-0003](../adr/0003-sealed-catalogues.md). An
`EventRegistry` will collect registrations through `listen()`, which takes an event type and a handler reference: either
an invokable class, or a class and a method. Its `seal()` will produce the `EventCatalogue` the dispatcher reads, and
registering a listener once the [registry](../GLOSSARY.md#registry) has been sealed will throw
`EventLifecycleException`.

```php
$registry->listen(ServerDeleted::class, PurgeServerFiles::class);
$registry->listen(ServerDeleting::class, [BlockDeletionWhileRunning::class, 'handle']);

$catalogue = $registry->seal();
```

Listeners will have no priority. They will run in the order they were registered, and nothing will reorder them.

A subscriber is a class that declares several listeners through attributes on its methods. Expanding one into
registrations means reading those attributes by reflection at registration time, which belongs to the
[module](../GLOSSARY.md#module) system. The dispatcher will have no concept of a subscriber.

### Dispatching

```php
public function dispatch(object $event): object;
```

The `EventDispatcher` will hand the event to each listener that matches it, one at a time and in registration order,
and then return the event. Anything the caller wants back, it will read from the event.

Each listener will be resolved through the dependency injection container, as
[RFC-0001](0001-dependency-injection-container.md) describes, the first time an event it matches is dispatched. A
[handler reference](../GLOSSARY.md#handler-reference) that cannot be resolved will therefore fail only then, with the
exception the container raises.

Listeners will have the `Process` [lifetime](../GLOSSARY.md#lifetime) from [RFC-0007](0007-binding-lifetimes.md), so one
instance will serve every dispatch for the life of the worker. A listener will hold no state belonging to a request, and
will reach anything that does through a [provider](../GLOSSARY.md#provider).

### Matching

A listener will match an event whose class is the type it was registered against, or extends or implements that type.
A listener registered against `object` will match every event. One listener registered against a marker interface will
therefore reach every event that implements it, including events a [module](../GLOSSARY.md#module) defines later.

Which listeners match cannot be worked out in advance. Because any object can be an event, the set of event classes is
open, and a class is not known until an instance of it is dispatched. The [match set](../GLOSSARY.md#match-set) for a
concrete class will be worked out on that class's first dispatch and kept for the life of the worker, so every later
dispatch of the class will read it directly.

### Cancellation

An event will opt into cancellation by implementing `Cancellable`:

```php
interface Cancellable
{
    public function cancel(string $reason): void;

    public function isCancelled(): bool;

    public function reason(): ?string;
}
```

Cancelling an event will stop propagation: no listener after the one that cancelled it will run. `dispatch()` will
still return the event, and the caller will read `isCancelled()` and `reason()` to learn whether it was cancelled and
why. Since propagation stops at the first cancellation, the reason the event carries will be the first one given.

### When a listener throws

An exception thrown by a listener will propagate. The dispatcher will catch nothing, and no listener after the one that
threw will run.

What that means is for the code that dispatched the event to handle. Work inside a transaction is rolled back by that
transaction. Code that dispatches an event inside something it must close, such as a [cycle](../GLOSSARY.md#cycle),
closes it in its own `try`/`finally`, so a listener that throws cannot skip the closing.

### Out of scope

- **Which events exist.** The code that dispatches an event decides what the event is and what it carries.
- **Subscribers and reading their attributes**, which belong to the [module](../GLOSSARY.md#module) system.
- **Durable delivery.** Delivering an event beyond the process, with an outbox to make delivery durable, will be a
  separate component built on this one.
- **An audit trail.** Also separate, and most likely a module.
- **Collecting contributions from modules.** A [collector](../GLOSSARY.md#collector) asks every module for contributions
  when a component chooses to; an event fires at a moment in a flow. They are different mechanisms.
- **Dispatching from the container.** Whatever owns a boundary, such as the driver that opens and closes a
  [cycle](../GLOSSARY.md#cycle), dispatches the events around it. The container never dispatches an event.

## Alternatives considered

This design rests on [ADR-0003](../adr/0003-sealed-catalogues.md), under which
mutable registries are sealed into immutable catalogues.

**Working out every match when the catalogue is sealed.** I rejected this because any object can be an event, so the
set of event classes is open and there is nothing complete to compute in advance. Working out the matches for each
concrete class on its first dispatch, and keeping them, saves the same work without needing the whole set.

## Backwards compatibility

Nothing will break. There is no event dispatcher today.

## Open questions

## Changelog

- 2026-10-05: Reworded in the readable house style, and the project's own vocabulary linked to the glossary. The
  design is unchanged.
- 2026-11-18: Rewrote the document in full, with no change to the design.

## Sources

- Issue [#68], Engine - Events, 2026-08-22, with no edits and no comments: a synchronous in-process dispatcher, any
  object being an event with naming carrying the meaning, events carrying whatever gathers contributions, listeners
  and subscribers, registration of handler references through a registry sealed into a catalogue, references rather
  than instances, `dispatch(object $event): object`, matching over class, parent and interface with `object` matching
  everything, matching resolved on first dispatch of a concrete class and kept for the life of the worker,
  `Cancellable` halting propagation, listeners resolved from the container as `Process` lifetime and reaching
  cycle-scoped state through a provider, a throwing listener propagating, and every boundary listed under Out of
  scope. Its tasks also include tests that listeners run in registration order and that a non-matching listener is
  neither resolved nor called.
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
