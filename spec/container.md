---
title: Container
includes: [ADR-0002, ADR-0004, RFC-0001]
---

# Container

The dependency injection container constructs objects and supplies their dependencies. A class is resolved from a
binding, or automatically from its constructor signature. Any callable, method or constructor is invoked with its
parameters supplied.

`Engine\Container\Container` is constructed with a resolver catalogue and a binding catalogue, and holds the shared
instances it resolves. It exposes `resolve()` and `invoke()`, and its binding catalogue is readable. It does not
implement PSR-11, per [ADR-0004](../adr/0004-the-container-does-not-implement-psr-11.md).

```php
$container = new Container($resolvers, $bindings);

$instance = $container->resolve(Resolution::for(Contract::class));
$result   = $container->invoke(Invocation::method($handler, 'handle'));
```

## Assembling the catalogues

`BindingCatalogue` and `ResolverCatalogue` are constructed directly from arrays. `BindingRegistry` and
`ResolverRegistry` collect registrations, but neither has a method that produces a catalogue, so nothing converts
one into the other. Tests construct catalogues themselves.

`BindingRegistry` is created for a module scope, and `bind()` returns a `BindingBuilder` carrying that scope.
Several builders accumulate under one abstract, which is how named and qualified child bindings sit alongside the
main binding. `ResolverRegistry` holds `default()` and `register()`.

## Bindings

| `BindingBuilder` method | Effect |
|---|---|
| `to(string\|object $concrete)` | A class-string sets the concrete class. An object sets the concrete class, as its own class, and the instance. |
| `as(string ...$aliases)` | Sets the aliases, replacing any set before. |
| `using(Closure $factory)` | Resolves the binding by invoking the factory through the container. |
| `named(string $name)` | Identifies the binding as the child selected by that name. |
| `qualifier(string $qualifier)` | Identifies the binding as the child selected by that qualifier attribute class. |
| `liminal()` | Holds the shared instance through a weak reference. |
| `lazily()` | Marks the binding lazy. |
| `notShared()` | Resolves a new instance every time. |

A builder's `shared` flag defaults to `true`. `Binding::from()` turns a builder into an immutable `Binding`, adding
the concrete class to the aliases when one is set. A `Binding` holds the abstract, concrete class, instance,
aliases, factory, a map of named children, a map of qualified children keyed by qualifier class, and the liminal,
lazy and shared flags. It answers `isBoundToInstance()` and `hasFactory()`.

### Looking one up

`BindingCatalogue::get(string $class, ?Named $named = null, ?Qualifier $qualifier = null)`:

1. The class is replaced by the abstract its alias points to. The alias map is consulted once, so an alias pointing
   at another alias is not followed.
2. With no binding for the class, the result is `null`.
3. With a name, the result is the named child binding, or `null`.
4. With a qualifier, the result is the child binding stored under the qualifier's class, or `null`.
5. Otherwise the binding itself.

The catalogue also holds a map of module scope to the classes bound under it. Nothing reads it.

## Resolving

`resolve(Resolution $resolution, bool $skipLazy = false)`:

1. **A cached instance** is returned if one is held for the resolution.
2. **A lazy proxy** is returned if the resolution asks to be lazy and lazy resolution is not being skipped.
3. **The binding** for the class, name and qualifier is looked up.
4. **The flags** are read: the instance and factory from the binding, `shared` from the binding or `false` when
   there is none, and liminality from the binding or the resolution.
5. **An instance or factory** on the binding produces the result. A concrete class replaces the class being
   resolved.
6. **Automatic construction** otherwise. A class carrying `NoResolution` throws. A class carrying `Lazy` is
   returned as a lazy proxy. A class that is not instantiable throws. A class with no constructor is instantiated
   directly, and one with a constructor has it invoked through the container. A class carrying `Liminal` makes the
   resolution liminal.
7. **Sharing.** A shared resolution is cached and returned; an unshared one is returned without being cached.

A class with a binding is shared unless the binding says otherwise. A class with no binding is not shared: `shared`
is read as `false` when there is no binding.

### Caching

Instances are held in four maps:

| Map | Key | Holds |
|---|---|---|
| Instances | Class | One instance per class. |
| Named instances | Class, then name | One instance per class and name. |
| Qualified instances | Class, then a list | Pairs of qualifier and instance. |
| Liminal instances | Class | A weak reference, returning the instance while it is alive. |

A qualified instance is found by walking the list for the class and comparing each entry's qualifier: its class
must match, and `equals()` must return true. `Qualifier` declares `equals(self $other): bool` for this.

A liminal instance is stored under its class alone, whether or not the resolution was named or qualified.

Reading and writing use different keys. A read looks under the class the resolution names; a write stores under the
binding's abstract where there is a binding. A shared binding resolved through an alias is therefore stored under
the abstract and looked for under the alias, and produces a new instance each time.

### Lazy proxies

A lazy resolution returns a proxy created with PHP's `newLazyProxy()`. When the proxy is first used, the container
resolves the same resolution with lazy resolution skipped, so a proxy's own initialisation cannot produce another
proxy.

### Ghost objects

`GhostResolver` handles the `Ghost` attribute. It takes the class from the dependency's type, and throws when the
type is absent or is not a class or interface. The binding's concrete class is used when there is one, and the
parameter's class otherwise. The object is created with `newLazyGhost()`, and its constructor is invoked through the
container when it is first used.

## Invoking

| `Invocation` factory | Invokes |
|---|---|
| `callable(callable $function)` | A callable. |
| `method(object\|string $on, string $method)` | A method on an object or a class. |
| `constructor(string\|object $class)` | A class's constructor, as `method($class, '__construct')`. |
| `with(array $arguments)` | Supplies arguments by parameter name, merged with any supplied before. |

A method must be public. A static method called with an object throws. A constructor called with no object
constructs a new instance. Any other method called with no object resolves the class through the container first.

Parameters are supplied in declaration order. A parameter whose name matches a supplied argument receives it. A
variadic parameter ends the list and is never supplied. Every other parameter becomes a `Dependency`.

## Dependencies

A `Dependency` carries the parameter's name, its type, whether it is optional, its `Named` attribute, its qualifier
attribute, its resolvable attribute, whether it has a default, that default, and whether it is liminal. The
qualifier and resolvable attributes are matched by interface, so any attribute implementing `Qualifier` or
`Resolvable` is found.

A dependency carrying both a name and a qualifier throws. A dependency carrying a resolvable attribute is resolved
by the resolver registered against that attribute's class, and by the default resolver otherwise. A resolver is
itself resolved through the container, lazily, the first time it is needed, and the same instance is used
afterwards.

### Resolving by type

`GenericResolver` is the default resolver:

- **A class or interface type** is resolved through the container, carrying over the dependency's name, qualifier
  and liminality. A named type that is neither a class nor an interface yields the default, or `null` where the type
  allows it, and otherwise throws.
- **An intersection type** needs a binding for at least one member type. Each is tried in turn, using its instance,
  resolving its concrete class, or invoking its factory. The first result satisfying every member type is used. A
  binding whose class cannot be reflected is skipped. With no result, the default is used, and otherwise it throws.
- **A union type** is resolved when exactly one member is a class, an interface or an intersection, by resolving
  that member. Otherwise the default is used, then `null` where the type allows it, and otherwise it throws.
- **No type** yields the default, and throws when there is none.

## Attributes

| Attribute | Declared for | Effect |
|---|---|---|
| `Named(string $name)` | Parameter | Resolves the dependency from the named binding. |
| A `Qualifier` attribute | Parameter | Resolves the dependency from the qualified binding. |
| `Lazy` | Class, parameter | On a class, the class resolves as a lazy proxy. |
| `Liminal` | Class, parameter | On a class, the class is held weakly. |
| `NoResolution` | Class, parameter | On a class, the class is never constructed automatically. |
| `Ghost` | Parameter | Resolves the dependency as a ghost object. |

`Lazy`, `Liminal`, `NoResolution` and `Ghost` each implement `Resolvable`. On a parameter, all four are therefore
found as the dependency's resolvable attribute, and resolution is handed to the resolver registered against that
attribute's class. `Ghost` has one. The other three resolve only where a resolver has been registered for them, and
otherwise throw `InvalidResolverException`. `Liminal` on a parameter also sets the dependency's liminal flag.

The three markers take effect as described above when they are read from a class.

## Errors

Every exception implements `ContainerException`.

| Exception | Thrown when |
|---|---|
| `UnresolvableClassException` | A class carrying `NoResolution` would be constructed automatically. |
| `NotInstantiableException` | A class to be constructed is not instantiable. |
| `DependencyResolutionException` | A dependency is both named and qualified, a type cannot be resolved, an intersection has no binding or none that satisfies it, a union cannot be resolved, or a ghost is asked for a type that is not a class. |
| `InvalidResolverException` | A resolvable attribute has no registered resolver. |
| `InvalidInvocationException` | An invocation is not callable, does not name a method, names a method that is not public, or calls a static method on an object. |
| `InvalidClassException` | A class cannot be reflected. |
| `InvalidMethodException` | A method cannot be reflected. |
| `MethodCallException` | Invoking a reflected method fails. |
| `InvalidFunctionException` | Reflecting a callable fails. The code marks this path unreachable. |
| `BindingNotFoundException` | Nothing throws it. |
