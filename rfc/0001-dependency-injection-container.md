---
id: RFC-0001
title: Dependency injection container
status: accepted
created: 2026-03-24
decided: 2026-03-28
backfilled: 2026-09-13
depends: [ADR-0002, ADR-0003, ADR-0004]
updates: []
obsoletes: []
---

# RFC-0001: Dependency injection container

## Abstract

A dependency injection container that constructs objects and supplies their dependencies. Classes are resolved
automatically from their constructor signatures, or through bindings that supply a concrete class, an instance or a
factory. Resolved instances are shared by default and can be held weakly, resolution can be deferred through lazy
proxies and ghost objects, and any callable, method or constructor can be invoked with its parameters supplied by the
container.

## Motivation

The panel needs a dependency injection solution. It needs to be:

- **Flexible**, without being convoluted, supporting the features expected of modern dependency injection.
- **Cacheable**, built so that bindings and resolution configuration can be compiled and cached for a production
  environment.
- **Easy to use**, requiring no in-depth understanding to get started.

## Proposal

### Concepts

| Term | Meaning |
|---|---|
| Resolution | Producing an instance of a class, from a binding or automatically from its constructor. |
| Invocation | Calling a callable, method or constructor with its parameters supplied by the container. |
| Abstract | The class or interface requested. |
| Concrete | The class instantiated in place of an abstract. |
| Binding | Registered resolution information for one abstract. |
| Alias | Another class name that resolves through the same binding. |
| Named binding | A child binding of an abstract, selected by a string name. |
| Qualified binding | A child binding of an abstract, selected by a qualifier attribute. |
| Shared | Resolved once, cached, and returned on every later resolution. |
| Liminal | Shared, but held through a weak reference, so it can be garbage collected once nothing else uses it. |
| Lazy proxy | An object that stands in for an instance not yet resolved, and forwards to that instance once it is. |
| Ghost object | An object of the final class left uninitialised, which becomes the instance itself once initialised. |
| Dependency | A parameter the container has to supply. |
| Resolver | An object that resolves a dependency. |
| Resolvable attribute | A parameter attribute that selects a custom resolver for that dependency. |

### Components

| Component | Responsibility |
|---|---|
| `Container` | Resolves classes and invokes callables. Holds the binding and resolver catalogues it is constructed with, and caches shared instances. |
| `Resolution` | Describes a request to resolve a class. |
| `Invocation` | Describes a callable, method or constructor to invoke, with any arguments supplied up front. |
| `Dependency` | Describes one parameter to supply: its name, type, default, and the attributes that affect its resolution. |
| `BindingRegistry`, `BindingBuilder` | Mutable. Collect binding definitions. |
| `Binding` | Immutable. The binding for one abstract, with its named and qualified child bindings. |
| `BindingCatalogue` | Immutable. The bindings and aliases the container is constructed with, looked up by class, name or qualifier. |
| `Resolver` | Contract for resolving a dependency. |
| `Resolvable` | Marker contract for attributes that select a custom resolver. |
| `ResolverRegistry` | Mutable. Collects the default resolver and the resolver for each resolvable attribute. |
| `ResolverCatalogue` | Immutable mapping of resolvable attribute to resolver, plus the default resolver, from which the container obtains resolver instances. |
| `GenericResolver` | The default resolver. |
| `GhostResolver` | The resolver for the `Ghost` attribute. |
| `Named`, `Qualifier` | Select named and qualified bindings for a parameter. |
| `Lazy`, `Liminal`, `NoResolution` | Attributes the container handles itself. |
| `Ghost` | A resolvable attribute, handled by `GhostResolver`. |
| `ContainerException` | Marker contract implemented by every exception the container throws. |

### Bindings

A binding is registered per abstract through a `BindingRegistry`. Each call to `bind()` returns a new
`BindingBuilder`; several builders for the same abstract accumulate, which is how named and qualified child
bindings sit alongside the main binding.

```php
$registry->bind(Contract::class)->to(Implementation::class);
$registry->bind(Contract::class)->to(new Implementation());
$registry->bind(Contract::class)->using(fn (Dependency $dependency) => new Implementation($dependency));
$registry->bind(Contract::class)->to(Implementation::class)->as(OtherContract::class);
$registry->bind(Contract::class)->named('secondary')->to(SecondaryImplementation::class);
$registry->bind(Contract::class)->qualifier(Archive::class)->to(ArchiveImplementation::class);
$registry->bind(Report::class)->notShared();
$registry->bind(Contract::class)->to(Implementation::class)->liminal();
$registry->bind(Contract::class)->to(Implementation::class)->lazily();
```

| Builder method | Effect |
|---|---|
| `to(string\|object $concrete)` | A class-string sets the concrete class. An object sets both the concrete class, as its class, and the instance. |
| `as(string ...$aliases)` | Sets the aliases, replacing any set before. When a concrete class is set, it is also an alias. |
| `using(Closure $factory)` | Resolves the binding by invoking the factory through the container, so the factory's own parameters are supplied. |
| `named(string $name)` | Identifies the binding as the child selected by that name. |
| `qualifier(string $qualifier)` | Identifies the binding as the child selected by that qualifier attribute class. |
| `liminal()` | Holds the shared instance weakly. |
| `lazily()` | Resolves the binding as a lazy proxy. |
| `notShared()` | Resolves a new instance every time. |

A `Binding` holds the abstract, the concrete class, the instance, the aliases, the factory, a map of named child
bindings keyed by name, a map of qualified child bindings keyed by qualifier class, and the liminal, lazy and shared
flags. Bindings are shared unless marked otherwise.

`BindingCatalogue::get(string $class, ?Named $named = null, ?Qualifier $qualifier = null)` looks a binding up:

1. An alias is replaced by the abstract it points to.
2. With no binding for the class, the result is `null`.
3. With a name, the result is the named child binding, or `null`.
4. With a qualifier, the result is the child binding for the qualifier's class, or `null`.
5. Otherwise, the result is the binding itself.

### Named and qualified bindings

Named bindings suit registries and managers, where resolved instances share a class but their contents depend on the
name. A dependency selects a named binding with the `Named` attribute, and a resolution with `named()`.

```php
public function __construct(
    #[Named('replica')] private Connection $connection,
) {}
```

Qualified bindings suit cases where more than a name is needed to resolve. A qualifier is an attribute class
implementing `Qualifier`, and the qualifier's class is the key: it selects the child binding, and any value the
attribute carries is ignored. The attribute is unique to one binding outcome, so one qualifier class cannot be reused
across different types. A dependency selects a qualified binding by carrying the attribute, and a resolution with
`qualifiedBy()`.

```php
#[Attribute(Attribute::TARGET_PARAMETER)]
final readonly class Archive implements Qualifier {}

public function __construct(
    #[Archive] private Storage $storage,
) {}
```

Shared instances of a qualified binding are cached per qualifier class. Selecting an instance by a value, such as a
region, is the job of a custom resolver. A dependency cannot be both named and qualified.

### Resolution

A resolution is requested with `Container::resolve()`, describing it with a `Resolution`:

```php
$instance = $container->resolve(Resolution::for(Contract::class));

$instance = $container->resolve(
    Resolution::for(Contract::class)
        ->with(['name' => 'value'])
        ->named('secondary')
        ->lazily(),
);
```

| `Resolution` method | Effect |
|---|---|
| `for(string $class)` | Starts a resolution for the class. |
| `with(array $arguments)` | Supplies constructor arguments by parameter name, merged with any supplied before. |
| `named(string $name)` | Selects the named binding. |
| `qualifiedBy(Qualifier $qualifier)` | Selects the qualified binding. |
| `resolveWith(Resolvable $resolvable)` | Resolves the class through the resolver paired with the resolvable attribute. |
| `lazily()` | Returns a lazy proxy. |
| `liminal()` | Holds the shared instance weakly. |

Resolving a class follows these steps:

1. **Cached instance.** If a shared instance is cached for the resolution, it is returned: by name for a named
   resolution, by qualifier class for a qualified one, from the weak cache for a liminal one while the instance is
   still alive, and otherwise by class.
2. **Lazy proxy.** If the resolution is lazy, a lazy proxy is returned, and the steps below run when it is first
   used.
3. **Binding.** The binding for the class, name and qualifier is looked up.
4. **Instance.** If the binding holds an instance, that instance is the result. If it has a factory, the factory is
   invoked through the container and its return value is the result. If it has a concrete class, that class is
   constructed in place of the abstract.
5. **Automatic construction.** Otherwise the class is constructed by the container. A class marked `NoResolution`
   cannot be, and throws. A class that is not instantiable throws. A class with no constructor is instantiated
   directly, and a class with one has its constructor invoked through the container, so its parameters are
   supplied as described under Invocation.
6. **Class attributes.** A class marked `Lazy` is always resolved as a lazy proxy, and a class marked `Liminal` is
   always held weakly, regardless of any binding or argument.
7. **Sharing.** Unless a binding marks it not shared, the instance is cached: weakly if liminal, by name or by
   qualifier where the resolution has one, and otherwise by class. The cache key is the binding's abstract, or the
   requested class where there is no binding.

When a lazy proxy is first used, it resolves the same resolution with lazy resolution skipped, so the proxy's own
initialisation cannot produce another proxy.

A resolution given a resolvable attribute with `resolveWith()` is resolved through the resolver paired with that
attribute, as a dependency carrying the attribute would be.

#### Lazy proxies

Lazy resolution uses the lazy objects introduced in PHP 8.4. A lazy proxy is always a proxy: when a property is first
accessed, the container resolves the real instance, and the proxy forwards to it from then on. A class with no
properties that could trigger initialisation is resolved straight away.

#### Ghost objects

A ghost is an object of the concrete class, created uninitialised. When it is first accessed, its constructor is
invoked on that same object through the container, and from then on it is the final instance in every meaningful way,
rather than a proxy in front of one. A dependency is resolved as a ghost with the `Ghost` attribute:

```php
public function __construct(
    #[Ghost] private ExpensiveService $service,
) {}
```

`GhostResolver` uses the binding's concrete class where there is one, and the parameter's class otherwise. A ghost
can only be created for a class or interface type.

### Invocation

Any callable, method or constructor can be invoked with its parameters supplied by the container, describing the
call with an `Invocation`:

```php
$result = $container->invoke(Invocation::callable($closure));
$result = $container->invoke(Invocation::method($object, 'handle'));
$result = $container->invoke(Invocation::method(Handler::class, 'handle')->with(['input' => $input]));
$object = $container->invoke(Invocation::constructor(Implementation::class));
```

| `Invocation` method | Effect |
|---|---|
| `callable(callable $function)` | Invokes a callable. |
| `method(object\|string $on, string $method)` | Invokes a method on an object or a class. |
| `constructor(string\|object $class)` | Invokes a class's constructor. |
| `with(array $arguments)` | Supplies arguments by parameter name, merged with any supplied before. |

Invoking follows these rules:

- **A callable** is called with its parameters supplied.
- **A method** must be public. A static method is called statically. A method on an object is called on that
  object. A method on a class-string is called on an instance of the class resolved through the container, except
  the constructor, which creates a new instance.

Parameters are supplied in declaration order:

1. A parameter with an argument supplied under its name receives that argument.
2. A variadic parameter ends the list. Variadic parameters are never supplied by the container.
3. Any other parameter becomes a `Dependency`, carrying its name, type, whether it is optional, its default, and its
   `Named`, qualifier, resolvable and `Liminal` attributes. The dependency is resolved by the resolver for its
   resolvable attribute if it has one, and by the default resolver otherwise.

### Resolvers

```php
interface Resolver
{
    public function resolve(Dependency $dependency, Container $container, array $arguments = []): mixed;
}
```

A `ResolverRegistry` sets the default resolver and registers a resolver against each resolvable attribute class. The
`ResolverCatalogue` the container is constructed with maps each resolvable attribute to its resolver. A resolver is
itself resolved through the container the first time it is needed, and the same instance is reused afterwards. A
dependency carrying a resolvable attribute with no registered resolver throws.

`Ghost` is the resolvable attribute defined here, paired with `GhostResolver`. Other components define their own
resolvable attributes and resolvers in the same way.

`GenericResolver`, the default, resolves a dependency by its type:

- **A class or interface type** is resolved through the container, carrying over the dependency's name, qualifier
  and liminal flag.
- **A scalar or other non-class type** receives its default value if it has one, or `null` if the type allows it,
  and throws otherwise.
- **An intersection type** needs a binding for at least one of its member types. Each such binding is tried in turn,
  using its instance, resolving its concrete class or invoking its factory, and the first result that satisfies every
  member type is used. Bindings whose concrete class does not exist are skipped. If none satisfies the type, the
  default value is used if there is one, and otherwise it throws.
- **A union type** is resolved when exactly one of its members is a class, an interface or an intersection, by
  resolving that member. Otherwise the default value is used if there is one, `null` if the type allows it, and it
  throws if neither.
- **No type** receives the default value if there is one, and throws otherwise.

### Attributes

| Attribute | Targets | Meaning |
|---|---|---|
| `Named(string $name)` | Parameter | Resolves the dependency from the named binding. |
| A `Qualifier` attribute | Parameter | Resolves the dependency from the qualified binding. |
| `Lazy` | Class, parameter | On a class, the class is always resolved as a lazy proxy. On a parameter, that dependency is resolved as a lazy proxy. |
| `Liminal` | Class, parameter | On a class, the class is always held weakly. On a parameter, that dependency is resolved liminally. |
| `NoResolution` | Class, parameter | On a class, the class is never constructed automatically: it must be bound with custom resolution logic, or have a factory bound to it. On a parameter, that dependency is never resolved automatically: it has to be supplied as an argument, and resolving it without one throws. |
| `Ghost` | Parameter | Resolves the dependency as a ghost object, through `GhostResolver`. |

`Lazy`, `Liminal` and `NoResolution` are handled by the container itself, as special cases of resolution, and do
not go through resolvers. `Ghost` is the first attribute to use the custom resolver mechanism.

### Caching

Shared instances live for the lifetime of the container:

- one instance per class
- one instance per class and name, for named resolutions
- one instance per class and qualifier class, for qualified resolutions
- a weak reference per class, for liminal resolutions, returning the instance only while it is still alive

Instances that are not shared are never cached. Resolver instances are cached by the resolver catalogue.

### Errors

Every exception implements `ContainerException`.

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

- **Assembling catalogues.** The container is constructed with a `BindingCatalogue` and a `ResolverCatalogue`.
  Building catalogues from registered builders belongs to the module lifecycle.
- **Compiling and caching** the catalogues for production.
- **Module scope.** A `BindingRegistry` is created for a module scope, and each binding records the scope it was
  registered under. What a scope means belongs to the module system.

## Alternatives considered

The decisions this design rests on are recorded separately, with the alternatives each one rejected:

- how a dependency selects its instance, through named and qualified bindings or a resolvable attribute paired with a
  resolver, in [ADR-0002](../adr/0002-dependencies-select-their-instance-through-parameter-attributes.md)
- sealing mutable registries into immutable catalogues, in
  [ADR-0003](../adr/0003-mutable-registries-are-sealed-into-immutable-catalogues.md)
- not implementing PSR-11, in [ADR-0004](../adr/0004-the-container-does-not-implement-psr-11.md)

No other alternatives were weighed.

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
