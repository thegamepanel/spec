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

We will add a synchronous, in-process event dispatcher in which any object can be an event. Listeners will be registered
against an event type by [handler reference](../GLOSSARY.md#handler-reference) and sealed into an immutable
[catalogue](../GLOSSARY.md#catalogue). A listener will receive every event of its type or a subtype of it, and an event
will be able to opt into cancellation, which stops any later listener from running.

## Motivation

Something that happens in one part of the panel often matters to another. When a server is deleted, whatever cleans
up after it needs to know, and so does whatever writes an audit trail, but the code deleting the server should not
have to know about either.

This matters more for a [module](../GLOSSARY.md#module), which has to react to what the [engine](../GLOSSARY.md#engine)
and other modules do without either side depending on the other.

## Proposal

### Events

Any object will be an event. There will be no marker interface to implement and nothing to extend: the event's class
is its identity.

The class name will carry the meaning. A name in the past tense, such as `ServerDeleted`, says the thing has happened
and cannot be stopped. A name in the present participle, such as `ServerDeleting`, says it is about to happen, and the
event may be cancellable.

An event that needs something back from its listeners will carry an object that collects it, such as a
[registry](../GLOSSARY.md#registry) or a builder, and listeners will call into that. The dispatcher will not gather or
combine anything itself.

### Registration

Registration will follow [ADR-0003](../adr/0003-mutable-registries-are-sealed-into-immutable-catalogues.md): an
`EventRegistry` will collect registrations, and its `seal()` method will produce the `EventCatalogue` the dispatcher
reads. Registering a listener after the [registry](../GLOSSARY.md#registry) has been sealed will throw.

```php
$registry->listen(ServerDeleted::class, PurgeServerFiles::class);
$registry->listen(ServerDeleting::class, [BlockDeletionWhileRunning::class, 'handle']);

$catalogue = $registry->seal();
```

| Method | Effect |
|---|---|
| `listen(string $event, string\|array $handler)` | Registers an invokable class, or a class and method, as a listener for the event type. |
| `seal(): EventCatalogue` | Produces the immutable catalogue and closes registration. |

A listener will be one handler for one event type, and the only thing the dispatcher stores and calls. A registration
will hold a [handler reference](../GLOSSARY.md#handler-reference), never an instance. A listener for an event that never
fires will never be constructed.

A subscriber is a class that declares several listeners through attributes on its methods. It will be expanded into
registrations by whatever reads those attributes. That is reflection over attributes at registration time, which belongs
to the [module](../GLOSSARY.md#module) system, and the dispatcher will have no concept of a subscriber.

### Dispatch

```php
public function dispatch(object $event): object;
```

The `EventDispatcher` will pass the event to each listener that matches it, one at a time and in the order the listeners
were registered, and then return the event. Anything the caller wants back, it will read from the event.

Each listener will be resolved through the dependency injection container, as
[RFC-0001](0001-dependency-injection-container.md) describes, the first time an event it matches is dispatched.
Listeners will have the `Process` [lifetime](../GLOSSARY.md#lifetime) from [RFC-0007](0007-binding-lifetimes.md). A
listener will hold no state belonging to a request, and will reach anything that does through a
[provider](../GLOSSARY.md#provider).

### Matching

A listener will match an event whose class is the type the listener is registered against, or extends or implements
that type. A listener registered against `object` will match every event.

One listener registered against a marker interface will therefore reach every event that implements it, including
events a [module](../GLOSSARY.md#module) defines later.

Matching cannot be worked out in advance. Because any object can be an event, the set of event classes is open, and a
class is not known until an instance of it is dispatched. The [match set](../GLOSSARY.md#match-set) for a concrete class
will be worked out on its first dispatch and kept for the life of the worker, so later dispatches of that class will
read it directly.

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

Cancelling an event will stop propagation: no listener after the one that cancelled it will run. `dispatch()` will still
return the event, and the caller will read `isCancelled()` and `reason()` to find out whether it was cancelled and why.
Because propagation stops at the first cancellation, the event will carry the first reason given.

### Listeners that throw

An exception thrown by a listener will propagate. The dispatcher will catch nothing, and no listener after the one that
threw will run.

What that means for the caller is the caller's concern. Work inside a transaction is rolled back by that
transaction. Code that dispatches an event inside something it must close, such as a [cycle](../GLOSSARY.md#cycle), does
the closing in its own `try`/`finally`, so a listener that throws cannot skip it.

### Errors

Registering a listener after the registry has been sealed will throw `EventLifecycleException`. A
[handler reference](../GLOSSARY.md#handler-reference) that cannot be resolved will fail when the listener is first
needed, with the exception the container raises, as [RFC-0001](0001-dependency-injection-container.md) describes.

### Out of scope

- **Which events exist.** Code that dispatches an event decides what the event is and what it carries.
- **Subscribers and reading their attributes.** Reading listener attributes off a class belongs to the
  [module](../GLOSSARY.md#module) system.
- **Listener priority.** Listeners will run in the order they were registered, and nothing will reorder them.
- **Durable delivery.** Delivering an event outside the process, with an outbox to make delivery durable, will be a
  separate component built on this one.
- **An audit trail.** Also separate, and most likely a module.
- **Collecting contributions from modules.** A collector asks every module for contributions when a component chooses
  to; an event fires at a moment in a flow. They are different mechanisms.
- **Dispatching from the container.** Whatever owns a boundary dispatches events around it. The container never
  dispatches an event.

## Alternatives considered

The decision this design rests on is recorded separately, with the alternatives it rejected:
[ADR-0003](../adr/0003-mutable-registries-are-sealed-into-immutable-catalogues.md), under which mutable registries are
sealed into immutable catalogues.

**Working out every match when the catalogue is sealed.** We rejected this because any object can be an event, so the
set of event classes is open and there is nothing complete to compute in advance. Working out the matches for each
concrete class on its first dispatch, and keeping them, saves the same work without needing the set in advance.

No other alternatives were weighed.

## Backwards compatibility

Nothing will break. There is no event dispatcher today.

## Open questions

## Changelog

- 2026-10-05: Reworded in the readable house style, and the project's own vocabulary linked to the glossary. The
  design is unchanged.

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
