---
id: RFC-0001
title: Dependency injection container
status: accepted
created: 2026-03-24
decided: 2026-03-28
backfilled: 2026-11-18
depends: [ADR-0002, ADR-0003, ADR-0004]
updates: []
obsoletes: []
---

# RFC-0001: Dependency injection container

## Abstract

I will give the panel a dependency injection container that constructs objects and supplies their dependencies. It will
resolve classes automatically from their constructors, or through [binding](../GLOSSARY.md#binding)s that supply a
[concrete](../GLOSSARY.md#concrete) class, an instance or a factory. Resolved instances will be
[shared](../GLOSSARY.md#shared) by default and can be held weakly, resolution can be deferred through a
[lazy proxy](../GLOSSARY.md#lazy-proxy) or a [ghost object](../GLOSSARY.md#ghost-object), and any callable, method or
constructor can be invoked with its parameters supplied by the container.

## Motivation

The panel needs a dependency injection solution, and it has to be:

- **Flexible**, without being convoluted, with the features expected of modern dependency injection.
- **Cacheable**, built so that [binding](../GLOSSARY.md#binding)s and resolution configuration can be compiled and
  cached for production.
- **Easy to use**, needing no in-depth understanding to get started.

## Proposal

### Bindings

A class with no binding will be constructed from its constructor. A [binding](../GLOSSARY.md#binding) holds the
resolution information for one [abstract](../GLOSSARY.md#abstract) that needs more than that. Bindings will be collected
through a `BindingRegistry`, where each call to `bind()` returns a new `BindingBuilder` for the abstract. The registry
and its builders will be mutable, and the `Binding` each builder describes, and the `BindingCatalogue` that holds them
for the container, will be immutable, as
[ADR-0003](../adr/0003-sealed-catalogues.md) decides. Several builders for one
abstract will accumulate, which is how [named binding](../GLOSSARY.md#named-binding)s and
[qualified binding](../GLOSSARY.md#qualified-binding)s sit alongside the main binding.

```php
$registry->bind(ServerRepository::class)->to(DatabaseServerRepository::class);
$registry->bind(ServerRepository::class)->to(new InMemoryServerRepository());
$registry->bind(NodeClient::class)->using(fn (HttpClient $http) => new NodeClient($http));
$registry->bind(ServerRepository::class)->to(DatabaseServerRepository::class)->as(ServerLookup::class);
$registry->bind(Connection::class)->named('replica')->to(ReplicaConnection::class);
$registry->bind(Storage::class)->qualifier(Archive::class)->to(ArchiveStorage::class);
$registry->bind(ServerReport::class)->notShared();
$registry->bind(NodeClient::class)->to(HttpNodeClient::class)->liminal();
$registry->bind(NodeClient::class)->to(HttpNodeClient::class)->lazily();
```

A binding first says how its instance is produced, where constructing the abstract itself would not do. `to()` given a
class-string sets the [concrete](../GLOSSARY.md#concrete) class to construct in place of the abstract. `to()` given an
object sets that object as the instance, and its class as the concrete class. Where construction needs logic of its own,
`using()` sets a factory, a closure the container invokes when the binding is resolved, supplying the factory's own
parameters as it would a constructor's.

A binding can also be reached through class names other than its abstract, its [alias](../GLOSSARY.md#alias)es, so that
code depending on any of them is given what the binding produces. `as()` sets them, replacing any set before. When a
concrete class is set, it is also an alias.

An abstract can have child bindings beside its main one, each selected by a name or by a
[qualifier](../GLOSSARY.md#qualifier) class, as described under choosing between bindings below. `named()` and
`qualifier()` make a builder's binding one of those children.

Last, a binding says how long its instance lives and when it is produced. A binding is [shared](../GLOSSARY.md#shared)
unless marked otherwise, so its instance is resolved once and returned on every later resolution. `notShared()` resolves
a new instance every time instead. `liminal()` keeps the instance shared but holds it weakly, so it can be garbage
collected once nothing else uses it. `lazily()` defers producing it, resolving the binding as a
[lazy proxy](../GLOSSARY.md#lazy-proxy).

The container will find a binding in the `BindingCatalogue` by class, and by name or qualifier where the resolution has
one. An alias is first replaced by the abstract it points to. With no binding for the class, the result is `null`.
Given a name, the result is the named child binding, or `null` if there is none. Given a qualifier, it is the child
binding for the qualifier's class, or `null`. Otherwise it is the binding itself. The lookup is
`BindingCatalogue::get()`, which takes the class with an optional `Named` attribute and an optional qualifier.

### Choosing between bindings of one type

A [named binding](../GLOSSARY.md#named-binding) suits registries and managers, where resolved instances share a class
but their contents depend on the name. A [dependency](../GLOSSARY.md#dependency) will select one with the `Named`
attribute, and a [resolution](../GLOSSARY.md#resolution) with `named()`.

```php
public function __construct(
    #[Named('replica')] private Connection $connection,
) {}
```

A [qualified binding](../GLOSSARY.md#qualified-binding) suits cases where more than a name is needed to resolve. A
[qualifier](../GLOSSARY.md#qualifier) is an attribute class implementing `Qualifier`, and the qualifier's class is the
key: it selects the child [binding](../GLOSSARY.md#binding), and any value the attribute carries is ignored. Because the
attribute is unique to one binding outcome, one qualifier class cannot be reused across different types. A dependency
will select a qualified binding by carrying the attribute, and a resolution with `qualifiedBy()`.

```php
#[Attribute(Attribute::TARGET_PARAMETER)]
final readonly class Archive implements Qualifier {}

public function __construct(
    #[Archive] private Storage $storage,
) {}
```

The [shared](../GLOSSARY.md#shared) instances of a qualified binding will be cached per qualifier class. Selecting an
instance by a value, such as a region, is the job of a custom [resolver](../GLOSSARY.md#resolver). A dependency cannot
be both named and qualified.

### Resolving a class

A [resolution](../GLOSSARY.md#resolution) will be requested with `Container::resolve()`, given a `Resolution` that
describes it:

```php
$repository = $container->resolve(Resolution::for(ServerRepository::class));

$client = $container->resolve(
    Resolution::for(NodeClient::class)
        ->with(['node' => $node])
        ->named('primary')
        ->lazily(),
);
```

A `Resolution` carries what the class alone does not say about one request. `for()` names the class. `with()` carries
constructor arguments by parameter name, merged with any supplied before, for values the caller holds, such as the node
a client is for. `named()` and `qualifiedBy()` select a [named binding](../GLOSSARY.md#named-binding) or a
[qualified binding](../GLOSSARY.md#qualified-binding), the same choice a [dependency](../GLOSSARY.md#dependency) makes
with an attribute. A resolution can also ask for what its binding does not impose: `lazily()` asks for a
[lazy proxy](../GLOSSARY.md#lazy-proxy), and `liminal()` asks for the [shared](../GLOSSARY.md#shared) instance to be
held weakly. Last, it can hand the request to a custom [resolver](../GLOSSARY.md#resolver): `resolveWith()`, given a
[resolvable attribute](../GLOSSARY.md#resolvable-attribute), resolves the class through the resolver paired with that
attribute, as a dependency carrying the attribute would be.

The container will resolve a class in these steps:

1. **Cached instance.** If a shared instance is cached for the resolution, it is returned: by name for a named
   resolution, by qualifier class for a qualified one, from the weak cache for a [liminal](../GLOSSARY.md#liminal) one
   while the instance is still alive, and otherwise by class.
2. **Lazy proxy.** If the resolution is lazy, a lazy proxy is returned, and the steps below run when it is first
   used.
3. **Binding.** The [binding](../GLOSSARY.md#binding) for the class, name and qualifier is looked up.
4. **Instance.** If the binding holds an instance, that instance is the result. If it has a factory, the factory is
   invoked through the container and its return value is the result. If it has a [concrete](../GLOSSARY.md#concrete)
   class, that class is constructed in place of the [abstract](../GLOSSARY.md#abstract).
5. **Automatic construction.** Otherwise the container constructs the class itself. A class marked `NoResolution`
   cannot be constructed this way, and throws. A class that is not instantiable throws. A class with no constructor
   is instantiated directly, and a class with one has its constructor invoked through the container, so its
   parameters are supplied as described under invocation below.
6. **Class attributes.** A class marked `Lazy` is always resolved as a lazy proxy, and a class marked `Liminal` is
   always held weakly, regardless of any binding or argument.
7. **Sharing.** Unless a binding marks it not shared, the instance is cached: weakly if liminal, by name or by
   qualifier where the resolution has one, and otherwise by class. The cache key is the binding's abstract, or the
   requested class where there is no binding.

When a lazy proxy is first used, it will resolve the same resolution with the lazy step skipped, so the proxy's own
initialisation cannot produce another proxy.

### Deferring construction

A [lazy proxy](../GLOSSARY.md#lazy-proxy) will use the lazy objects introduced in PHP 8.4, and will always stay a proxy:
when a property is first accessed, the container resolves the real instance, and the proxy forwards to it from then on.
A class with no properties that could trigger initialisation will be resolved straight away instead.

A [ghost object](../GLOSSARY.md#ghost-object) is an object of the [concrete](../GLOSSARY.md#concrete) class itself,
created uninitialised. When it is first accessed, its constructor will be invoked on that same object through the
container, and from then on it is the final instance in every meaningful way, rather than a proxy in front of one. A
[dependency](../GLOSSARY.md#dependency) will be resolved as a ghost with the `Ghost` attribute:

```php
public function __construct(
    #[Ghost] private NodeInventory $inventory,
) {}
```

`GhostResolver` will create the ghost for the [binding](../GLOSSARY.md#binding)'s concrete class where there is one, and
for the parameter's class otherwise. A ghost can only be created for a class or interface type.

### Invoking with supplied parameters

Any callable, method or constructor will be invocable with its parameters supplied by the container, given an
`Invocation` that describes the call:

```php
$result = $container->invoke(Invocation::callable($closure));
$result = $container->invoke(Invocation::method($handler, 'handle'));
$result = $container->invoke(Invocation::method(StartServer::class, 'handle')->with(['server' => $server]));
$client = $container->invoke(Invocation::constructor(HttpNodeClient::class));
```

`with()` supplies arguments by parameter name, merged with any supplied before. A callable is called with its
parameters supplied. A method must be public. A static method is called statically, and a method on an object is
called on that object. A method named on a class-string is called on an instance of the class resolved through the
container, except the constructor, which creates a new instance.

Parameters will be supplied in declaration order. A parameter with an argument supplied under its name receives that
argument. A variadic parameter ends the list, because the container never supplies variadic parameters. Any other
parameter becomes a [dependency](../GLOSSARY.md#dependency), carrying its name, type, whether it is optional, its
default, its `Named` attribute, its [qualifier](../GLOSSARY.md#qualifier), its
[resolvable attribute](../GLOSSARY.md#resolvable-attribute) and its `Liminal` attribute. The dependency is resolved by
the [resolver](../GLOSSARY.md#resolver) paired with its resolvable attribute if it has one, and by the default resolver
otherwise.

### Resolvers

A [resolver](../GLOSSARY.md#resolver) will resolve one [dependency](../GLOSSARY.md#dependency):

```php
interface Resolver
{
    public function resolve(Dependency $dependency, Container $container, array $arguments = []): mixed;
}
```

A [resolvable attribute](../GLOSSARY.md#resolvable-attribute) is an attribute implementing the `Resolvable` marker
contract. A `ResolverRegistry` will set the default resolver and register a resolver against each
resolvable attribute class, and the `ResolverCatalogue` the container is
constructed with will map each resolvable attribute to its resolver. A resolver is itself resolved through the container
the first time it is needed, and the same instance is reused afterwards. A dependency carrying a resolvable attribute
with no registered resolver throws.

`Ghost`, paired with `GhostResolver`, is the resolvable attribute defined here. Other components will define their
own resolvable attributes and resolvers in the same way.

`GenericResolver`, the default, will resolve a dependency by its type:

- **A class or interface type** is resolved through the container, carrying over the dependency's name,
  [qualifier](../GLOSSARY.md#qualifier) and [liminal](../GLOSSARY.md#liminal) flag.
- **A scalar or other non-class type** receives its default value if it has one, or `null` if the type allows it,
  and throws otherwise.
- **An intersection type** needs a [binding](../GLOSSARY.md#binding) for at least one of its member types. Each such
  binding is tried in turn, using its instance, resolving its [concrete](../GLOSSARY.md#concrete) class or invoking its
  factory, and the first result that satisfies every member type is used. Bindings whose concrete class does not exist
  are skipped. If none satisfies the type, the default value is used if there is one, and otherwise it throws.
- **A union type** is resolved when exactly one of its members is a class, an interface or an intersection, by
  resolving that member. Otherwise the default value is used if there is one, `null` if the type allows it, and it
  throws if neither.
- **No type** receives the default value if there is one, and throws otherwise.

### Attributes the container handles itself

`Lazy`, `Liminal` and `NoResolution` will be special cases of [resolution](../GLOSSARY.md#resolution), handled by the
container itself rather than by [resolver](../GLOSSARY.md#resolver)s, and each can be placed on a class or on a
parameter. On a class, `Lazy` always resolves the class as a [lazy proxy](../GLOSSARY.md#lazy-proxy), and on a parameter
it resolves that [dependency](../GLOSSARY.md#dependency) as one. On a class, `Liminal` always holds the class weakly,
and on a parameter it resolves that dependency [liminal](../GLOSSARY.md#liminal)ly. On a class, `NoResolution` means the
class is never constructed automatically: it must be bound with custom resolution logic, or have a factory bound to it.
On a parameter, it means that dependency is never resolved automatically and has to be supplied as an argument, and
resolving it without one throws.

`Named` and [qualifier](../GLOSSARY.md#qualifier) attributes go on parameters only, as does `Ghost`, which is the first
attribute to use the custom resolver mechanism.

### Shared instances

The container will keep [shared](../GLOSSARY.md#shared) instances for its own lifetime. There will be one instance per
class, one per class and name for named resolutions, and one per class and qualifier class for qualified resolutions. A
[liminal](../GLOSSARY.md#liminal) resolution will keep a weak reference per class, returning the instance only while it
is still alive. Instances that are not shared are never cached. Each [resolver](../GLOSSARY.md#resolver) instance is
cached by the resolver catalogue.

### Errors

Every exception the container throws will implement `ContainerException`.

| Exception | Thrown when |
|---|---|
| `UnresolvableClassException` | A class or dependency marked `NoResolution` would be resolved automatically. |
| `NotInstantiableException` | A class to be constructed is abstract, an interface, or otherwise not instantiable. |
| `DependencyResolutionException` | A dependency cannot be resolved: a type with no default that does not allow `null`, an unsatisfiable intersection or union type, a ghost requested for a non-class type, or a dependency both named and qualified. |
| `InvalidResolverException` | A dependency's resolvable attribute has no registered resolver. |
| `InvalidInvocationException` | A method to be invoked is not public, or an invocation is not callable or does not name a method. |
| `InvalidClassException`, `InvalidMethodException` | A class or method cannot be reflected. |
| `MethodCallException` | Calling a reflected method fails. |

### Out of scope

- **Assembling catalogues.** The container is constructed with a `BindingCatalogue` and a `ResolverCatalogue`. Building
  the [catalogue](../GLOSSARY.md#catalogue)s from registered builders belongs to the [module](../GLOSSARY.md#module)
  lifecycle.
- **Compiling and caching** the catalogues for production.
- **Module scope.** A `BindingRegistry` is created for a [module scope](../GLOSSARY.md#module-scope), and each
  [binding](../GLOSSARY.md#binding) records the scope it was registered under. What a scope means belongs to the module
  system.

## Alternatives considered

This design rests on three decisions recorded separately, each with the alternatives it rejected:
[ADR-0002](../adr/0002-parameter-attributes.md), on how a
[dependency](../GLOSSARY.md#dependency) selects its instance through named and qualified bindings or a
[resolvable attribute](../GLOSSARY.md#resolvable-attribute) paired with a [resolver](../GLOSSARY.md#resolver);
[ADR-0003](../adr/0003-sealed-catalogues.md), on sealing each mutable
[registry](../GLOSSARY.md#registry) into an immutable [catalogue](../GLOSSARY.md#catalogue); and
[ADR-0004](../adr/0004-no-psr-11.md), on not implementing PSR-11.

## Backwards compatibility

Nothing breaks. This is the first component, and the repository has no source code yet.

## Open questions

## Changelog

## Sources

- Issue [#21], Engine - Dependency Injection: the goals in its original text of 2026-03-24, and the features added
  later the same day, in its edit history. `created` is its date.
- Resolution and invocation, the handling of `Lazy`, `Liminal` and `NoResolution` by the container rather than by
  resolvers, and ghosts as the first custom resolver: part of the original plan but not written into the issue, and
  first written down on 2026-09-13.
- A qualifier's class being the key with any value ignored, `resolveWith()` resolving through the paired resolver,
  `Lazy` and `NoResolution` applying on parameters, and class attributes applying regardless of binding: first
  written down on 2026-09-14. The code at [728ad64] differs on each of these points, and the design here is what was
  intended.
- The code at [728ad64] also differs from this design:
  - a resolution's arguments are never read, so `with()` supplies nothing to a constructor
  - an instance is cached only where a binding exists, because `shared` is read as false without one, so an
    auto-wired class is constructed again on every resolution
  - `Liminal` on a parameter is routed to a resolver, as `Lazy` and `NoResolution` are, because all three implement
    `Resolvable`
  - `Qualifier` declares `equals()`, which the container calls when matching a cached qualified instance and which
    the qualifier example above does not implement
  - a binding does not carry the module scope its builder was created for, which the binding catalogue holds
    separately
  - `BindingNotFoundException` and `InvalidFunctionException` also implement `ContainerException` and are absent
    from the errors above, and nothing throws the first
- PR [#23], feat(container): Dependency Injection, merged 2026-03-28: the implementation, whose merge is `decided`.
  Its commits land on `main` individually, ending at [728ad64]. The API, the resolution and invocation steps, the
  resolver behaviour and the exceptions are described from
  [`src/Container` at 728ad64](https://github.com/thegamepanel/panel/tree/728ad64/src/Container) and
  [`tests/Unit/Container` at 728ad64](https://github.com/thegamepanel/panel/tree/728ad64/tests/Unit/Container).
- Commit [7fad8cd], the initial commit: placeholder files under `src/` and no source code.

[#21]: https://github.com/thegamepanel/panel/issues/21
[#23]: https://github.com/thegamepanel/panel/pull/23
[728ad64]: https://github.com/thegamepanel/panel/commit/728ad64
[7fad8cd]: https://github.com/thegamepanel/panel/commit/7fad8cd
