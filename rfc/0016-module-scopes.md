---
id: RFC-0016
title: Module scopes
status: proposed
created: 2026-09-16
decided:
depends: [ADR-0002, ADR-0020, RFC-0001, RFC-0007, RFC-0015]
updates: []
obsoletes: []
---

# RFC-0016: Module scopes

## Abstract

I will make a [resolution](../GLOSSARY.md#resolution) know which [module](../GLOSSARY.md#module)'s code it is
constructing. The container will keep a stack of [module scope](../GLOSSARY.md#module-scope)s, pushing a
[binding](../GLOSSARY.md#binding)'s [owner](../GLOSSARY.md#owner) while that binding resolves and popping it afterwards,
so that everything constructed beneath it inherits the owner by recursion, and it will stamp the scope in force onto
each [dependency](../GLOSSARY.md#dependency) and [invocation](../GLOSSARY.md#invocation) as it builds them, so that
factories and resolvers receive it without any signature changing. This design is the mechanism alone: what a module is
then handed belongs to module resources.

## Motivation

A [module](../GLOSSARY.md#module) needs things that belong to it rather than to the [engine](../GLOSSARY.md#engine),
such as its own logger channel, its own filesystem root, or a query builder bound to its own schema. Each of those needs
one question answered at the moment something is constructed: which module is this for?

Nothing can answer it today. A module's [registrar](../GLOSSARY.md#registrar) is instantiated directly, before the
container exists, as [RFC-0015](0015-module-system.md) describes. The container cannot change once it is built. Nothing
at dispatch knows which module a class belongs to. There is no current module held anywhere at runtime.

The [binding](../GLOSSARY.md#binding) is what carries the identity. A binding registered through a module's
`EngineBuilder` records the [module identifier](../GLOSSARY.md#module-identifier) that registered it, as
[RFC-0015](0015-module-system.md) describes, so resolving that binding is the point at which the container can know
whose code it is about to construct.

## Proposal

### The scope stack

The container will hold the [scope stack](../GLOSSARY.md#scope-stack) itself, and will push onto it, pop from it, and
stamp what it builds with the [module scope](../GLOSSARY.md#module-scope) on top. It will be a stack rather than a
single value because [resolution](../GLOSSARY.md#resolution) nests: a scoped [binding](../GLOSSARY.md#binding) may
depend on another [module](../GLOSSARY.md#module)'s scoped binding, and both scopes are live at once.

Every push will be paired with its pop in a `try`/`finally`. Otherwise a constructor throwing part way down a graph
would leave the stack dirty for the rest of the worker's life. The [resolution stack](../GLOSSARY.md#resolution-stack)
in [RFC-0006](0006-container-improvements.md) avoids the same failure by removing its entry however the resolution ends.

Whatever state the stack is in, it will be emptied when a [cycle](../GLOSSARY.md#cycle) closes, through the
[disposal](../GLOSSARY.md#disposal) in [RFC-0007](0007-binding-lifetimes.md). It will be emptied per cycle rather than
per request because the cycle is the container's own unit of work: HTTP opens one for each request, a queue worker in a
warm process opens one for each job, and a scheduler opens one for each tick. A scope leaked in a long-lived process
would show up as one module's channel appearing in another module's cycles, which is hard to trace back to its cause.

### Ownership

A [binding](../GLOSSARY.md#binding) registered through a module-scoped `EngineBuilder` will carry its
[owner](../GLOSSARY.md#owner), the [module identifier](../GLOSSARY.md#module-identifier) that registered it. A binding
the [engine](../GLOSSARY.md#engine) registers itself will carry none. That presence will be the whole signal: when the
container resolves a binding with an owner, it will push the owner for the length of the
[resolution](../GLOSSARY.md#resolution) and pop it afterwards, and when it resolves one without, it will push nothing.

There will be no flag to opt a binding into being aware of its scope. Such a flag would be true exactly when an owner
exists, so it would carry no information. Nor will anything gate reading a [module scope](../GLOSSARY.md#module-scope):
the scope will be stamped unconditionally, so every resolver and factory will already have it.

The container will therefore have one behaviour rather than a set of modes: push when a binding has an owner, and pop
when it is done.

### Capturing a scope

A [module scope](../GLOSSARY.md#module-scope) will be captured when a [dependency](../GLOSSARY.md#dependency) or an
[invocation](../GLOSSARY.md#invocation) is created, and held, rather than looked up when it is read. A nested
[resolution](../GLOSSARY.md#resolution) will move the stack between a value being built and being used, and several live
values holding different scopes will be the expected state rather than an edge case.

The two will acquire it differently, because they are built differently. A `Dependency` is constructed with all its
values, so it will take the scope as one more of them, alongside those
[RFC-0001](0001-dependency-injection-container.md) gives it, and whoever builds it will supply the scope. An
`Invocation` is built by static factories that hold no reference to the container and so can read nothing; the container
will stamp the scope onto it afterwards, as it already does with the arguments supplied to it.

Neither will inherit a scope from whatever caused it. A dependency stamped `backups` may resolve to a
[binding](../GLOSSARY.md#binding) owned by `servers`, at which point the container will push `servers`, and the
invocation that constructs it will be stamped `servers`. Propagating the scope from the parent would be wrong in exactly
the case that matters.

### Using a scope

A [module scope](../GLOSSARY.md#module-scope) will reach the code that needs it along two paths, and neither will change
a published signature.

A factory will receive an [invocation](../GLOSSARY.md#invocation), which will carry the scope. Most first-party scoped
services will be factories, because whatever manages them will track their instances itself and the container will not
cache them.

A constructor parameter will become a [dependency](../GLOSSARY.md#dependency), which will carry the scope, and will be
handed to a [resolver](../GLOSSARY.md#resolver). `Resolver::resolve()` is a contract third parties implement, as
[ADR-0002](../adr/0002-parameter-attributes.md) records, and it will stay as
it is: the scope will arrive inside the dependency it already receives.

### Scope and attributes

An attribute carries information about one place a dependency is used. A [module scope](../GLOSSARY.md#module-scope)
carries information about one [module](../GLOSSARY.md#module). Neither will stand in for the other, and both will apply
at once.

A logging channel attribute will show how the two combine. Logging is not designed yet, and whatever designs it will
ship the attribute and its resolver with no scope to combine with, because modules do not exist yet. Once they do, the
resolver will read the name on the attribute relative to the scope:

| Site | Attribute | Scope | Channel |
|---|---|---|---|
| Engine component | `#[Channel('database')]` | none | `database` |
| Engine component | `#[Channel]` | none | the engine's own |
| Module class | `#[Channel]` | `backups` | `backups` |
| Module class | `#[Channel('audit')]` | `backups` | `backups.audit` |

A module will never write its own [module identifier](../GLOSSARY.md#module-identifier) where it consumes something.
That will be the property this design protects: a module's code will not name the module.

### Naming another module's scope

The `Scoped` attribute will give one parameter an [explicit scope](../GLOSSARY.md#explicit-scope), resolving it under a
named [module scope](../GLOSSARY.md#module-scope) rather than the [ambient scope](../GLOSSARY.md#ambient-scope). It will
exist for a [module](../GLOSSARY.md#module) that extends another module and needs that module's instance of something,
such as the servers module's [connection](../GLOSSARY.md#connection):

```php
public function __construct(
    #[Scoped(Servers::IDENT)] private Connection $servers,
) {}
```

This will open no hole. Modules are PHP running in one process with no isolation between them, so a module wanting
another module's connection can already construct one directly. The attribute will make explicit, and searchable, an
intent that would otherwise be invisible.

It will stamp that one [dependency](../GLOSSARY.md#dependency) and push nothing. A parameter attribute that silently
changed the scope of an arbitrarily deep graph would be hard to reason about, and what it will be meant for is factories
at the leaves, with no dependencies of their own, so the difference rarely shows. If a [binding](../GLOSSARY.md#binding)
with an [owner](../GLOSSARY.md#owner) turns up during that [resolution](../GLOSSARY.md#resolution), it will push its
owner as any other would. Pushing can be added later, where it could not be taken away.

The attribute will carry two constraints:

- **It takes a constant, never a literal.** The constant will be covered by the check under Rules below, so a typo will
  be fatal rather than a wrong instance.
- **The named scope must be a [module dependency](../GLOSSARY.md#module-dependency) the module declares.** Coupling
  between modules belongs in the [manifest](../GLOSSARY.md#manifest) rather than buried in a constructor, and declaring
  it will mean a missing target is found when the module boots rather than when something resolves.

There will be no attribute for the opposite case, a parameter opting out of the
ambient scope. `Unmodular` will be the name reserved for it, because every shorter word
is already taken. `Unscoped` already means something else for a `#[Collect]` method in
[RFC-0015](0015-module-system.md), `Platform` collides with the [panel context](../GLOSSARY.md#panel-context) of that
name, `Core` is ambiguous when core features are themselves modules, as
[ADR-0013](../adr/0013-core-modules.md) records, and `Global` reads badly in PHP. It is named here so that
it is not renamed later, and will be added when a parameter needs it.

### Where a scope is pushed

A [module scope](../GLOSSARY.md#module-scope) will be pushed in four places, each taking the
[module identifier](../GLOSSARY.md#module-identifier) from what it already holds:

- resolving a [binding](../GLOSSARY.md#binding) with an [owner](../GLOSSARY.md#owner), from the binding;
- dispatching a route to its action, from the owner the collecting handler recorded when it collected the route;
- a queue worker running a job, from the job's payload, where it carries one;
- the register and boot phases, from the [manifest](../GLOSSARY.md#manifest), which whatever drives the module lifecycle
  already holds.

Anything resolved outside these will run with no module scope. That gap will be accepted rather than closed.

### An empty stack

Nothing will throw when the [scope stack](../GLOSSARY.md#scope-stack) is empty. Whether an empty stack matters will
belong to the service rather than to the container, because services will differ. A logger will tolerate it: no
[module scope](../GLOSSARY.md#module-scope) will mean the [engine](../GLOSSARY.md#engine)'s own channel, and the loss
will be cosmetic. A query builder bound to a schema will not: no scope will mean querying the wrong tables, silently. So
the schema builder's factory will check the scope and throw, and the container will stay ignorant of which services care
about being unscoped.

In debug, a class under a [module](../GLOSSARY.md#module)'s autoload prefix resolved with an empty stack will raise a
warning. That will show up a missing push site while a module is being developed, and will cost nothing in production.

### Memoisation

Stamping a [module scope](../GLOSSARY.md#module-scope) will cost nothing, because a
[dependency](../GLOSSARY.md#dependency) is not memoised and there is no cached structure for a scope to invalidate, as
[ADR-0020](../adr/0020-no-parameter-memoisation.md) records.

A cached resolution plan would change that. Every entry in one would depend on the scope, so a plan could not be shared
between resolutions running under different scopes. Nothing proposes such a plan and the module lifecycle has no compile
step, so this is recorded as a conflict to be aware of rather than something given up.

### Rules

- **When one module resolves a binding another module registered, the scope will be the registering module's.** The
  [binding](../GLOSSARY.md#binding)'s code will be that [module](../GLOSSARY.md#module)'s code, so its logger, its
  schema and its filesystem will be the right ones.
- **A module's declared identifier constant will be checked against its manifest at discovery.** A module whose two
  disagree will fail to load. That will turn drift from a silently wrong schema into a module that refuses to start, and
  will be what lets the `Scoped` attribute take a constant safely.
- **Access across modules will be policed by capabilities, in the components that care.** Routing will do this with
  `CrossRoutes`, as [RFC-0015](0015-module-system.md) describes. The container will not be an enforcement layer.

### Out of scope

- **Module resources.** What a [module](../GLOSSARY.md#module) declares, what is provisioned for it and what it is
  handed. This design will only make a [resolution](../GLOSSARY.md#resolution) able to say whose code it is.
- **Enforcement.** The container will not decide whether one module may reach another's things.
- **The `Unmodular` attribute.** Named above so that it is not renamed, and added when a parameter needs it.
- **Pushing from the `Scoped` attribute.**
- **Logging.** The channel attribute will be an example, not a design.
- **Thread safety.** The open question below covers it.

## Alternatives considered

This design rests on [ADR-0002](../adr/0002-parameter-attributes.md),
dependencies selecting their instance through parameter attributes, and
[ADR-0020](../adr/0020-no-parameter-memoisation.md), resolution data not being memoised per
parameter.

**A single current scope rather than a stack.** I rejected this because a scoped [binding](../GLOSSARY.md#binding) can
depend on another [module](../GLOSSARY.md#module)'s scoped binding, so the value has to nest.

**A flag marking a binding as aware of its scope.** I rejected this because the flag would be true exactly when an
[owner](../GLOSSARY.md#owner) exists, and so would say nothing the owner does not.

**Looking the scope up when a value is read, rather than capturing it when the value is built.** I rejected this because
a nested [resolution](../GLOSSARY.md#resolution) moves the stack in between, so the answer read later is not the one
that was true when the value was created.

**Inheriting a scope from the value that caused it.** I rejected this because a dependency stamped with one module
resolving to a binding owned by another must produce the second, which is precisely the case inheritance would get
wrong.

**Pushing a scope from the `Scoped` attribute.** I rejected this because a parameter attribute that silently changes the
scope of an arbitrarily deep graph is hard to reason about. Pushing can be added later if a case needs it, and could not
be taken back.

**Throwing on an empty stack.** I rejected this because services disagree about whether an empty stack is a problem, so
the decision belongs to each service rather than to the container.

## Backwards compatibility

- `Dependency` will take one more constructor argument, so everything that builds one will supply the scope.
- `Invocation` will carry a scope the container stamps, which its static factories do not supply.

Nothing else will change. `Resolver::resolve()` will be untouched, and no binding needs editing: a
[binding](../GLOSSARY.md#binding) with no [owner](../GLOSSARY.md#owner) will behave exactly as it does today.

## Open questions

### Whether queue workers run as an in-process thread pool

A [scope stack](../GLOSSARY.md#scope-stack) on a shared container is mutable state across concurrent resolutions, and
interleaved pushes and pops would corrupt it. A container per thread makes the question moot; a shared container needs
the stack to be thread-local. Nothing in the record says workers are threaded, and the only concurrency anywhere in it
is concurrent fragment requests and concurrent index creation. It belongs to the bootstrap design, where the driver
contract lives.

## Changelog

- 2026-10-05: Reworded in the readable house style, and the project's own vocabulary linked to the glossary. The
  design is unchanged.
- 2026-11-18: Rewrote the document in full, with no change to the design.

## Sources

- Issue [#70], Engine - Container - Module scopes, 2026-09-03, edited three minutes later the same day: that there is no
  ambient current module and why, the binding being the identity carrier, the scope stack and why it is a stack,
  `try`/`finally` and the dirty stack it prevents, emptying at the cycle boundary rather than the request boundary and
  the leak it avoids, ownership as the whole signal with no opt-in flag and no gate on reading, the empty stack
  belonging to the service with the logger and schema builder as the two cases, the debug warning, capture at creation
  with the two values acquiring it differently and neither inheriting, consumption through factories and resolvers with
  no signature changed, scope and attributes being orthogonal with the channel table, `#[Scoped]` with its two
  constraints and why it pushes nothing, `Unmodular` and the four names it was chosen over, the push sites and the
  accepted residue, the three rules, the memoisation conflict, every boundary, and the thread-pool question.
- Issue [#39], Engine - Modules - Collectors, 2026-06-02, with no edits: `Unscoped` already meaning something different
  for a `#[Collect]` method, which is why the opposite case needs another name.
- Issue [#47], Engine - Container - Circular dependency detection, original text of 2026-08-08 in its edit history:
  popping only on the success path leaving stale entries after any failure, which is the precedent for wrapping push and
  pop.
- Issue [#67], Engine - Sessions, 2026-08-22, and issue [#59], Engine - Database - Schema: tables, indexes and DDL
  objects, 2026-08-19: concurrent fragment requests and concurrent index creation, the only concurrency anywhere in the
  record, which is why the thread-pool question is open rather than answered.

[#39]: https://github.com/thegamepanel/panel/issues/39
[#47]: https://github.com/thegamepanel/panel/issues/47
[#59]: https://github.com/thegamepanel/panel/issues/59
[#67]: https://github.com/thegamepanel/panel/issues/67
[#70]: https://github.com/thegamepanel/panel/issues/70
