---
id: RFC-0018
title: Ghost resolution
status: proposed
created: 2026-11-19
decided:
depends: [ADR-0002, ADR-0029]
updates: [RFC-0001, RFC-0006, RFC-0007]
obsoletes: []
---

# RFC-0018: Ghost resolution

## Abstract

I will have the container resolve a [dependency](../GLOSSARY.md#dependency) marked with `Ghost` as a
[ghost resolution](../GLOSSARY.md#ghost-resolution) with its own step, and remove `GhostResolver`. A
[ghost object](../GLOSSARY.md#ghost-object) will be created only where the container would construct the instance
itself. Elsewhere, and for a class with no properties that could trigger initialisation, a ghost resolution will be
resolved as every other [resolution](../GLOSSARY.md#resolution) is.

## Motivation

A [dependency](../GLOSSARY.md#dependency) marked with `Ghost` is resolved by `GhostResolver`. `Ghost` is the first
[resolvable attribute](../GLOSSARY.md#resolvable-attribute) in the container, and `GhostResolver` is the
[resolver](../GLOSSARY.md#resolver) paired with it. `GhostResolver` creates a
[ghost object](../GLOSSARY.md#ghost-object) of the [binding](../GLOSSARY.md#binding)'s
[concrete](../GLOSSARY.md#concrete) class where the binding has one, and of the parameter's class otherwise. It creates
the ghost object without a [resolution](../GLOSSARY.md#resolution).

A ghost object does not select an instance. The dependency's type, name or [qualifier](../GLOSSARY.md#qualifier) has
already selected the instance. A ghost object changes only the time at which the instance is constructed. The ghost
object is created first, and its constructor runs when the ghost object is first accessed.

Under [RFC-0007](0007-binding-lifetimes.md), a ghost object is given a [lifetime](../GLOSSARY.md#lifetime).
`GhostResolver` decides the lifetime. Where the [instance cache](../GLOSSARY.md#instance-cache) selected for that
lifetime holds an instance, `GhostResolver` injects that instance. Otherwise `GhostResolver` caches the ghost object,
and adds the ghost object to the [disposal list](../GLOSSARY.md#disposal-list) when the ghost object is initialised.
Deciding a lifetime, reading and writing an instance cache and adding to the disposal list are steps of the container's
resolution. A resolver can call only the container's public methods.

A binding that holds an instance has the instance's class as its concrete class. A dependency marked with `Ghost`
receives a new ghost object of that class, and the instance is not used. For a binding that holds a factory, the
dependency receives a ghost object of the binding's concrete class where the binding has one. Otherwise it receives a
ghost object of the parameter's class. The factory is not used either way. Under [RFC-0007](0007-binding-lifetimes.md),
a ghost object of a `Process` or `Cycle` lifetime is also cached. Every later resolution of the binding then receives
the ghost object in place of the instance or the factory's result.

PHP creates a ghost object of a class with no properties that could trigger initialisation as already initialised. It
does not call the initialiser, so the constructor never runs.

## Proposal

### Requesting a ghost resolution

`Ghost` will no longer be a [resolvable attribute](../GLOSSARY.md#resolvable-attribute). It will become one of the
attributes the container handles itself, as `Lazy`, `Liminal` and `NoResolution` are under
[RFC-0001](0001-dependency-injection-container.md). `Ghost` will still go on parameters only.

A [dependency](../GLOSSARY.md#dependency) made for a parameter marked with `Ghost` will record the marking in a `ghost`
property, as it records `Liminal` in `liminal`. `ghost` will default to `false`, as `liminal` does. The default
[resolver](../GLOSSARY.md#resolver) will make a [ghost resolution](../GLOSSARY.md#ghost-resolution) for a dependency of
a class or interface type. It will carry over the name, [qualifier](../GLOSSARY.md#qualifier) and
[liminal](../GLOSSARY.md#liminal) flag, as it does for a class or interface type under
[RFC-0001](0001-dependency-injection-container.md).

Resolving a parameter marked with `Ghost` whose type is not a class or interface will throw
`DependencyResolutionException`. Scalar, union and intersection types are examples.
[RFC-0001](0001-dependency-injection-container.md) specifies the same exception for a
[ghost object](../GLOSSARY.md#ghost-object) of such a type. A parameter that is also marked with `NoResolution` will
never be resolved automatically, as [RFC-0001](0001-dependency-injection-container.md) describes. `Ghost` will have no
effect on such a parameter. Resolving a parameter marked with `Ghost` that is also marked with a resolvable attribute
will throw `DependencyResolutionException`. Under [RFC-0001](0001-dependency-injection-container.md), the container
throws the same for a dependency that is both named and qualified. `Lazy`, `Liminal` and `NoResolution` are not
resolvable attributes, as [RFC-0001](0001-dependency-injection-container.md) decides. A parameter marked with `Ghost`
and one of them will therefore not throw. A parameter marked with both `Ghost` and `Lazy` will be resolved as a ghost
resolution that, under [RFC-0001](0001-dependency-injection-container.md), also asks for a
[lazy proxy](../GLOSSARY.md#lazy-proxy). The ghost step will come first, so the lazy proxy step will apply only where
the [resolution](../GLOSSARY.md#resolution) continues from the ghost step. `ghost()` will throw
`InvalidResolutionException` when the resolution already has a resolvable attribute, and `resolveWith()` will throw it
when the resolution is already a ghost resolution. Under [RFC-0006](0006-container-improvements.md), `named()` and
`qualifiedBy()` throw the same way for a resolution given both a name and a qualifier.

`GhostResolver` will be removed. `ProviderResolver`, which [RFC-0007](0007-binding-lifetimes.md) registers alongside it,
will be registered without it. `Resolution` will gain two methods, alongside `lazily()`:

| Method | Effect |
|---|---|
| `ghost(): self` | Marks the resolution as a ghost resolution. Throws `InvalidResolutionException` when the resolution has a resolvable attribute. |
| `isGhost(): bool` | Reports whether the resolution is a ghost resolution. |

### The ghost step

The [resolution](../GLOSSARY.md#resolution) steps of [RFC-0006](0006-container-improvements.md) will gain a ghost step
between the cached instance step and the [lazy proxy](../GLOSSARY.md#lazy-proxy) step. A resolution's
[lifetime](../GLOSSARY.md#lifetime) is decided alongside its [effective liminality](../GLOSSARY.md#effective-liminality)
in those steps, as [RFC-0007](0007-binding-lifetimes.md) describes. A
[ghost resolution](../GLOSSARY.md#ghost-resolution) will reach the ghost step having replaced an
[alias](../GLOSSARY.md#alias) and looked up its [binding](../GLOSSARY.md#binding). The lookup will throw where a name or
[qualifier](../GLOSSARY.md#qualifier) finds no binding, as [ADR-0029](../adr/0029-selection-requires-binding.md)
decides. The ghost resolution will also have decided its lifetime. Deciding the lifetime will throw for a `Cycle`
lifetime with no [cycle](../GLOSSARY.md#cycle) open, or while one is closing. The ghost resolution will have found no
instance in the [instance cache](../GLOSSARY.md#instance-cache) selected for it. The ghost step will apply these rules
in order:

1. If the binding holds an instance or a factory, the resolution will continue to the next step.
2. If `NoResolution` applies, the resolution will throw, as the lazy proxy step does.
3. The class of the [ghost object](../GLOSSARY.md#ghost-object) will be the binding's
   [concrete](../GLOSSARY.md#concrete) class, or the class from the alias step where there is no concrete class. If that
   class cannot be created as a ghost object, or cannot be constructed by the construction step, the resolution will
   throw `NotInstantiableException`. Interfaces, classes declared `abstract`, classes extending an internal PHP class
   other than `stdClass` and classes whose constructor is not public are examples.
4. If that class has no properties that could trigger initialisation, the resolution will continue to the next step. PHP
   would create a ghost object of that class already initialised.
5. Otherwise a ghost object of that class will be created and returned. It will be put at once in the instance cache
   selected for the resolution. The entry will be keyed by the class from the alias step, and by the name or qualifier
   class where the resolution has one. A `Transient` lifetime selects no cache, so the ghost object will not be cached.

A resolution that continues from the ghost step will go on through the lazy proxy step and the steps after it. A
[lazy resolution](../GLOSSARY.md#lazy-resolution) will be given a lazy proxy at the lazy proxy step.

A ghost resolution whose binding or class is lazy will be given a ghost object wherever rule 5 applies. Under
[RFC-0006](0006-container-improvements.md), such a resolution is given a lazy proxy at the lazy proxy step.
[RFC-0001](0001-dependency-injection-container.md) states that a class marked `Lazy` is always resolved as a lazy proxy.
Both statements will hold only where the resolution continues from the ghost step.

A ghost resolution that returns a ghost object will add no entry to the
[resolution stack](../GLOSSARY.md#resolution-stack). A circular dependency through that ghost resolution will therefore
not throw. A resolution can continue from the ghost step and be given no lazy proxy. A ghost resolution of a binding
that holds a factory, or of a class with no properties that could trigger initialisation, is an example. That resolution
will add its entry at the resolution stack step, as every other resolution does. A circular dependency through it will
throw. [RFC-0006](0006-container-improvements.md) states that `Ghost` breaks a circular dependency. That statement will
hold only where a ghost object is created. [RFC-0006](0006-container-improvements.md) states that a ghost object is
created without a resolution. A ghost object will instead be created by a ghost resolution, at the ghost step.

Once a ghost object is cached, every resolution of the same entry will receive it at the cached instance step. A
resolution that is not a ghost resolution will also receive the ghost object. A circular dependency through such a
resolution will not throw when the ghost object is received. It will throw when the ghost object is first accessed while
its own class is still being constructed eagerly. The ghost object's initialiser will then resolve a class that is still
on the resolution stack.

### Initialisation

A [ghost object](../GLOSSARY.md#ghost-object) will be initialised when it is first accessed. The container will then
invoke the constructor of its class, where there is one, with the [resolution](../GLOSSARY.md#resolution)'s arguments,
as the construction step does. The ghost object will become the instance itself, as
[RFC-0001](0001-dependency-injection-container.md) describes. Initialisation will add no entry to the
[resolution stack](../GLOSSARY.md#resolution-stack). The [dependencies](../GLOSSARY.md#dependency) the ghost object's
constructor resolves will add their own entries. Once the constructor has run, the ghost object will be added to the
[disposal list](../GLOSSARY.md#disposal-list) after those dependencies, as [RFC-0007](0007-binding-lifetimes.md)
describes for a constructed instance. A ghost object that is never initialised will never be constructed, and never
disposed of.

A ghost object can be created while its own class is being constructed eagerly. A class that depends on a second class,
whose constructor takes the first as a ghost object, is an example. The ghost object will then be created while the
construction step resolves the first class's constructor arguments. Once those arguments are resolved, the construction
step will look in the [instance cache](../GLOSSARY.md#instance-cache) selected for the resolution. If it finds an
uninitialised ghost object there, it will initialise that ghost object with those arguments instead of constructing a
new object. The constructor will run once, inside the ghost object's initialiser. The sharing step will then cache the
ghost object, and everything holding it will hold the instance itself. The ghost object will be added to the disposal
list once, when its constructor has run, as every other ghost object is. If the construction step finds a ghost object
that is already initialised, it will return that ghost object without running the constructor again. A
[ghost resolution](../GLOSSARY.md#ghost-resolution) with arguments, made while one of the class's constructor arguments
is being constructed, is an example. The arguments the construction step resolved will not be used.

The construction of that class can throw before the constructor runs. A constructor argument that cannot be resolved,
after the ghost object was created, is an example. The ghost object will then stay in the instance cache, uninitialised.
Every later resolution of the same entry will receive the ghost object, and its constructor will run when it is first
accessed.

The construction step will use the ghost object only where the ghost resolution selected the same entry in the same
instance cache. A `Transient` [lifetime](../GLOSSARY.md#lifetime) selects no cache. A [liminal](../GLOSSARY.md#liminal)
resolution and a resolution that is not liminal occupy separate entries, as [RFC-0006](0006-container-improvements.md)
decides. For a `Transient` lifetime or a different liminality, the construction step will construct a new object, and
the ghost object will stay a separate instance.

A ghost object whose constructor throws will stay uninitialised in the instance cache that holds it. Its constructor
will run again when it is next accessed, so everything holding the ghost object will keep the same object. A ghost
object of a `Cycle` lifetime will be initialised in the [cycle](../GLOSSARY.md#cycle) that is open when the ghost object
is first accessed. [RFC-0007](0007-binding-lifetimes.md) describes the same for a ghost object held by a `Process`
object. The ghost object will be added to that cycle's disposal list. Initialising it while a cycle is closing, or with
no cycle open, will throw `LifetimeException`, as resolving a `Cycle` lifetime does in either state.

A resolution can reach a cached ghost object while the ghost object's constructor is running. A resolution made during
that constructor, for a class that depends on the ghost object's class, is an example. That resolution will receive the
ghost object, as every other holder does. Code that uses the ghost object before its constructor finishes will find it
partly constructed.

A [lazy proxy](../GLOSSARY.md#lazy-proxy) whose resolution finds an uninitialised ghost object in the instance cache
will initialise the ghost object before using it. A ghost object created after the lazy proxy, by a ghost resolution of
the same entry, is an example. PHP does not accept an uninitialised ghost object as the instance behind a lazy proxy.

## Alternatives considered

This design rests on two decisions recorded separately. [ADR-0002](../adr/0002-parameter-attributes.md) decides that a
[resolvable attribute](../GLOSSARY.md#resolvable-attribute) is paired with a [resolver](../GLOSSARY.md#resolver) where a
[dependency](../GLOSSARY.md#dependency)'s instance is selected at runtime.
[ADR-0002](../adr/0002-parameter-attributes.md) names [ghost object](../GLOSSARY.md#ghost-object)s as the first use of
that pairing in the container. A ghost object selects no instance, so that decision does not apply to it. `Ghost` will
no longer be paired with a resolver. [ADR-0029](../adr/0029-selection-requires-binding.md) decides that a name or
[qualifier](../GLOSSARY.md#qualifier) that finds no [binding](../GLOSSARY.md#binding) throws.

**Keeping the `Ghost` and `GhostResolver` pairing.** The resolver would build a
[ghost resolution](../GLOSSARY.md#ghost-resolution) from the dependency's class, name and qualifier, and pass it to the
container. The resolver could instead call a public container method that runs the container's
[lifetime](../GLOSSARY.md#lifetime), cache and [disposal](../GLOSSARY.md#disposal) steps for a ghost object. I rejected
this because the resolver would decide nothing. A ghost object selects no instance, and every step that selects one
stays in the container. In either form, the resolver would only forward the dependency it was given. The public method
would also add to the container's public API for one resolver to call.

**Refusing a ghost resolution of a binding that holds a factory.** The container would throw, naming `Lazy` as the
alternative. I rejected this because a binding that holds a factory gives the container no constructor to run, so the
ghost step has nothing to defer. Resolving the ghost resolution as every other [resolution](../GLOSSARY.md#resolution)
is gives the dependency the factory's result. That result is the instance the dependency's type, name or qualifier
selected.

**Resolving a ghost resolution of a binding that holds a factory as a lazy resolution.** Where a lazy proxy can be
created, the factory would run when the [lazy proxy](../GLOSSARY.md#lazy-proxy) is first accessed. I rejected this
because the dependency would receive a lazy proxy rather than the factory's result.

**Putting the lazy proxy step before the ghost step.** A ghost resolution would be given a lazy proxy even where a ghost
object could be created. A ghost resolution whose binding or class is lazy, or that also asks for a lazy proxy, is an
example. I rejected this because a parameter marked with `Ghost` is more specific than a binding or class that is lazy.
The ghost step already continues to the lazy proxy step wherever it creates no ghost object.

**Letting a resolvable attribute take precedence over `Ghost`.** The resolver would produce the instance, and `Ghost`
would be ignored. I rejected this because a dependency marked with `Ghost` would then neither receive a ghost object nor
throw.

**Creating a ghost object for a class whose constructor is not public.** The ghost object would be created, and would
throw when it is first accessed. I rejected this because rule 3 of the ghost step applies only where the container would
construct the class itself. A constructor that is not public is then certain to fail. Throwing at once reports the
failure when the dependency is resolved, not wherever the ghost object is first accessed. The lazy proxy step of
[RFC-0006](0006-container-improvements.md) makes no such check, and a lazy proxy of such a class still fails when it is
first accessed.

**Removing a ghost object from the cache when its constructor throws.** A later resolution would create a new ghost
object. I rejected this because anything already holding the failed ghost object would keep it. A `Process` or `Cycle`
lifetime would then have two objects for one entry.

**Initialising a ghost object of a `Cycle` lifetime with no cycle open, without disposing of it.** The ghost object
would be constructed and left off every [disposal list](../GLOSSARY.md#disposal-list). I rejected this because a
resource its constructor opens would never be released.

**Refusing to initialise a ghost object of a `Cycle` lifetime outside the cycle that created it.** Each
[cycle](../GLOSSARY.md#cycle) would have an identity. A ghost object initialised in any later cycle would throw. I
rejected this because, under [RFC-0007](0007-binding-lifetimes.md), a `Cycle` instance held by a `Process` object can
already be used in a later cycle. Under [RFC-0007](0007-binding-lifetimes.md), the same holds for a lazy proxy of such
an instance. Ghost objects would then behave differently from lazy proxies.

**Throwing where a ghost resolution finds its own entry on the resolution stack.** The ghost step would throw
`CircularDependencyException` instead of creating a ghost object. I rejected this because a circular dependency through
`Ghost` would then throw whenever the ghost object's own class was resolved first.

**Leaving a ghost object created during its own class's construction as a separate instance.** The construction step
would cache the object it constructs in place of the ghost object. Anything holding the ghost object would keep it. I
rejected this because a cached entry of a `Process` or `Cycle` lifetime would then have two objects. The constructor
would also run twice, once on the object constructed and once on the ghost object when it is first accessed.

**Treating a ghost object reached during its own initialisation as a circular dependency.** The cached instance step
would throw for a ghost object still initialising. I rejected this because the cached instance step would also refuse a
dependency that only stores the ghost object. Storing a ghost object before it is constructed is the purpose of `Ghost`.
Under [RFC-0006](0006-container-improvements.md), a lazy proxy throws when it is first accessed while the construction
it stands in for is still on the [resolution stack](../GLOSSARY.md#resolution-stack). Using the lazy proxy inside the
constructor that requested it is an example. The container cannot do the same for a ghost object, because PHP does not
run a ghost object's initialiser again while it is running. The container therefore has no point at which to detect the
use.

**Throwing where a lazy proxy's resolution finds an uninitialised ghost object.** The lazy proxy would throw
`DependencyResolutionException` when it is first accessed. Under [RFC-0006](0006-container-improvements.md), a lazy
proxy throws the same for an instance PHP does not accept. I rejected this because two dependencies of the same entry
would then work or fail depending on which of them was resolved first.

## Backwards compatibility

These will change for code that resolves through the container today:

- `GhostResolver` will be removed, and `Ghost` will no longer be a
  [resolvable attribute](../GLOSSARY.md#resolvable-attribute). Today a parameter marked with `Ghost` throws
  `DependencyResolutionException` unless `GhostResolver` is registered, with `InvalidResolverException` as its previous
  exception.
- A [ghost resolution](../GLOSSARY.md#ghost-resolution) of a [binding](../GLOSSARY.md#binding) that holds an instance or
  a factory will receive what every other [resolution](../GLOSSARY.md#resolution) of the binding produces. It will be
  given the instance or the factory's result, or a [lazy proxy](../GLOSSARY.md#lazy-proxy) where it is a
  [lazy resolution](../GLOSSARY.md#lazy-resolution). A circular dependency through a binding that holds a factory will
  throw where no lazy proxy is given. Today, for a binding that holds an instance, the
  [dependency](../GLOSSARY.md#dependency) receives a new [ghost object](../GLOSSARY.md#ghost-object) of the instance's
  class. For a binding that holds a factory, the ghost object is of the binding's [concrete](../GLOSSARY.md#concrete)
  class, or of the parameter's class where there is none.
- For a ghost resolution of a class with no properties that could trigger initialisation, the class will be constructed
  eagerly. A circular dependency through that class will throw. Today the ghost object is created already initialised,
  so the class's constructor never runs. A circular dependency through that class does not throw.
- A ghost resolution of a class to which `NoResolution` applies will throw where no binding holds an instance or a
  factory for it. Today a ghost object of the class is created.
- A ghost resolution of a class that cannot be created as a ghost object will throw `NotInstantiableException`. An
  interface or a class declared `abstract` with no concrete class, and a class extending an internal PHP class other
  than `stdClass`, are examples. Today `DependencyResolutionException` is thrown, with PHP's `Error` as its previous
  exception.
- A ghost resolution of a class whose constructor is not public will throw `NotInstantiableException`. Today the ghost
  object is created, and throws `InvalidInvocationException` when it is first accessed.
- A ghost resolution will replace an [alias](../GLOSSARY.md#alias), as every other resolution does. Today, where the
  class the alias names has no concrete class, the ghost object is created for the alias itself.
- A ghost object will be cached under its [lifetime](../GLOSSARY.md#lifetime) and its
  [effective liminality](../GLOSSARY.md#effective-liminality), as every other resolved instance is. Under
  [RFC-0007](0007-binding-lifetimes.md), a ghost object is already cached under its lifetime. Today every ghost object
  is a separate instance, never cached, and `GhostResolver` ignores whether the dependency is
  [liminal](../GLOSSARY.md#liminal).
- A ghost object created while its own class is being constructed eagerly will become the instance that construction
  produces. The ghost object will become that instance only where both resolutions select the same entry in the same
  [instance cache](../GLOSSARY.md#instance-cache). A `Transient` lifetime selects no cache. Today the dependency holds a
  second object of the class, and the constructor runs on it when it is first accessed.
- A parameter marked with `Ghost` that is also marked with a resolvable attribute will throw
  `DependencyResolutionException`. Today the first of the two attributes written is taken as the parameter's resolvable
  attribute. A parameter marked with `Ghost` that is also marked with `Lazy`, `Liminal` or `NoResolution` will follow
  the rules under Requesting a ghost resolution. Today `GhostResolver` gives it a ghost object, and the other attribute
  has no effect.
- `resolveWith()` will no longer accept `Ghost`, and `ghost()` will request a ghost resolution instead. Today
  `resolveWith()` accepts `Ghost`, and the container ignores it, as it ignores every attribute given to `resolveWith()`.

## Open questions

## Changelog
