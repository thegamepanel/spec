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

A resolution knows whose code it is constructing. The container holds a stack of module scopes, pushing a binding's
owner while that binding resolves and popping afterwards, so everything constructed beneath it inherits the owner by
recursion. The active scope is stamped onto each `Dependency` and `Invocation` as it is built, so factories and
resolvers receive it without any signature changing. This is the mechanism alone: what a module is then handed
belongs to module resources.

## Motivation

A module needs things that belong to it rather than to the engine: its own logger channel, its own filesystem root,
a query builder bound to its own schema. Each of those needs an answer to one question, asked at the moment
something is constructed: which module is this for.

Nothing can answer it today. Registrars are instantiated directly, before the container exists, per
[RFC-0015](0015-module-system.md). The container is immutable once built. Nothing at dispatch knows which module a
class belongs to. There is no ambient current module at runtime.

The binding is what carries the identity. A binding registered through a module's `EngineBuilder` records the ident
that registered it, per [RFC-0015](0015-module-system.md), so resolving that binding is the point at which the
container can know whose code it is about to construct.

## Proposal

### Concepts

| Term | Meaning |
|---|---|
| Scope | The module a resolution is running under, named by its ident. |
| Owner | The module that registered a binding. |
| Scope stack | The scopes currently pushed, innermost last. |
| Ambient scope | The scope at the top of the stack, which a resolution runs under. |
| Explicit scope | A scope named at a parameter, used instead of the ambient one. |

### Components

| Component | Responsibility |
|---|---|
| `Container` | Holds the scope stack, pushes and pops it, and stamps the active scope onto what it builds. |
| `Binding` | Carries the ident that registered it, or none for the engine's own bindings. |
| `Dependency`, `Invocation` | Carry the scope they were built under. |
| `Scoped` | Attribute naming another module's scope for one dependency. |

### The scope stack

The container holds a stack rather than a single value, because resolution nests: a scoped binding may depend on
another module's scoped binding, and both scopes are live at once.

Push and pop are wrapped in `try`/`finally`. A constructor throwing part way down a graph would otherwise leave the
stack dirty for the remaining life of the worker, which is the same failure the resolution stack in
[RFC-0006](0006-container-improvements.md) avoids by removing its entry however the resolution finishes.

The stack is emptied when a cycle closes, whatever state it is in, through the disposal in
[RFC-0007](0007-binding-lifetimes.md). A cycle rather than a request, because the vocabulary is the container's: HTTP
opens one per request, a queue worker in a warm process opens one per job, and a scheduler opens one per tick. A
scope leaked in a long-lived process shows up as one module's channel appearing in another module's cycles, which is
miserable to trace back to its cause.

### Ownership

A binding registered through a module-scoped `EngineBuilder` has an owning ident. That presence is the entire
signal:

- A binding with an owner pushes it for the duration of the resolution, and pops afterwards.
- A binding with no owner, which is every binding the engine registers, pushes nothing.

There is no flag to opt a binding into being scope-aware. Such a flag would be true exactly when an owner exists, so
it would carry no information. There is no gate on reading one either: the scope is stamped unconditionally, so
every resolver and factory already has it.

The container therefore has one behaviour rather than a set of modes: push when a binding has an owner, pop when it
is done.

### Capturing a scope

A scope is captured when a `Dependency` or an `Invocation` is created, and held, rather than looked up when it is
read. A nested resolution moves the stack between a value being built and being consumed, and several live values
holding different scopes is the expected state, not an edge case.

The two acquire it differently, because they are built differently:

| | How it acquires the scope |
|---|---|
| `Dependency` | Constructed with its values, so it takes the scope as one more of them, alongside the parameter, type, name, qualifier, resolvable attribute, default and liminality of [RFC-0001](0001-dependency-injection-container.md). Whoever builds it supplies the scope. |
| `Invocation` | Built by static factories that have no container reference and so can read nothing. The container stamps it after the fact, as it already does for the arguments supplied to it. |

Neither inherits a scope from whatever caused it. A `Dependency` stamped `X` may resolve to a binding owned by `Y`,
at which point the container pushes `Y`, and the `Invocation` that constructs it is stamped `Y`. Propagating from
the parent would be wrong in exactly the case that matters.

### Consuming a scope

Two paths, and neither changes a published signature:

- **A factory** receives an `Invocation`, which carries the scope. Most first-party scoped services are factories,
  because whatever manages them owns their instance tracking and the container does not cache them.
- **A constructor parameter** becomes a `Dependency`, which carries the scope, and is handed to a `Resolver`.
  `Resolver::resolve()` is a contract third parties implement, per
  [ADR-0002](../adr/0002-dependencies-select-their-instance-through-parameter-attributes.md), and it is untouched:
  the scope arrives inside the `Dependency` it already receives.

### Scope and attributes

An attribute carries information about one use site. A scope carries information about one module. Neither
substitutes for the other, and both apply at once.

A channel attribute is the worked example. Logging is not designed yet, and whatever designs it ships the attribute
and its resolver with no scope to compose against, because modules do not exist. Once they do, the resolver composes
the two, and the name on the attribute is read relative to the scope:

| Site | Attribute | Scope | Channel |
|---|---|---|---|
| Engine component | `#[Channel('database')]` | none | `database` |
| Engine component | `#[Channel]` | none | the engine's own |
| Module class | `#[Channel]` | `backups` | `backups` |
| Module class | `#[Channel('audit')]` | `backups` | `backups.audit` |

A module never writes its own ident at a consumption site. That is the property worth protecting: a module's code
does not name the module.

### Naming another module's scope

`#[Scoped]` marks one dependency as resolving under a named scope rather than the ambient one. The case it exists
for is a module extending another module and needing that module's instance of something.

```php
public function __construct(
    #[Scoped(Servers::IDENT)] private Connection $servers,
) {}
```

This opens no hole. Modules are PHP running in one process with no isolation, so a module wanting another module's
connection can already construct one directly. The attribute makes an intent explicit and greppable that would
otherwise be invisible.

It stamps that one `Dependency` and pushes nothing. A parameter attribute silently rescoping an arbitrarily deep
subtree is hard to reason about, and what this targets is leaf factories with no onward dependencies, so the
difference rarely shows. If a binding with an owner turns up during that resolution, ordinary ownership pushing
applies as usual. Pushing can be added later; it could not be removed.

Two constraints:

- **It takes a constant, never a literal.** A constant inherits the drift validation below, so a typo is fatal
  rather than a wrong instance.
- **The named scope must be a declared dependency of the module.** Cross-module coupling belongs in the manifest
  rather than buried in a constructor, and declaring it means a missing target is found when the module boots rather
  than when something resolves.

There is no attribute for the opposite case, a parameter opting out of the ambient scope. `Unmodular` is the name
reserved for it, because every shorter word is already taken: `Unscoped` means something else for a `#[Collect]`
method in [RFC-0015](0015-module-system.md), `Platform` collides with the panel context, `Core` is ambiguous when
core features are themselves modules, per
[ADR-0013](../adr/0013-core-features-are-modules.md), and `Global` reads badly in PHP. It is named here so it is not
renamed later, and added when a parameter actually needs it.

### Where a scope is pushed

| Site | Ident from |
|---|---|
| Resolving a binding with an owner | The binding. |
| Router and action dispatch | The owning ident the collector handler recorded when it collected the route. |
| A queue worker | The job's payload, if it carries one. |
| Register and boot | The manifest, which the lifecycle driver already holds. |

Anything resolved outside these runs with no scope. That residue is accepted rather than closed.

### An empty stack

Nothing errors when the stack is empty. Whether that matters belongs to the service, not to the container, because
services differ:

- A logger tolerates it. No scope means the engine's own channel, and the loss is cosmetic.
- A schema-bound query builder does not. No scope means querying the wrong tables, silently.

So the schema builder's factory checks the scope and throws, and the container stays ignorant of which services have
opinions about being unscoped.

In debug, a class resolved under a module's autoload prefix with an empty stack warns, which surfaces a missing push
site while developing at no cost in production.

### Memoisation

Stamping a scope costs nothing, because a `Dependency` is not memoised and there is no cached structure for a scope
to invalidate, per [ADR-0020](../adr/0020-resolution-data-is-not-memoised-per-parameter.md).

A cached resolution plan would change that. Every entry in one would be scope-dependent, so a plan could not be
shared between resolutions running under different scopes. Nothing proposes such a plan and the module lifecycle has
no compile step, so this is recorded as a conflict to be aware of rather than something given up.

### Rules

- **When module `Y` resolves a binding registered by module `X`, the scope is `X`.** The binding's code is `X`'s
  code, so `X`'s logger, schema and filesystem are the right answers.
- **A module's declared ident constant is validated against its manifest at discovery**, and a module whose two
  disagree fails to load. That turns drift from a silently wrong schema into a module that refuses to start, and it
  is what lets `#[Scoped]` take a constant safely.
- **Cross-module access is policed by capabilities, in the components that care**, as routing does with
  `CrossRoutes`, per [RFC-0015](0015-module-system.md). The container is not an enforcement layer.

### Out of scope

- **Module resources.** What a module declares, what is provisioned for it and what it is handed. This design only
  makes a resolution able to say whose code it is.
- **Enforcement.** The container does not decide whether one module may reach another's things.
- **`#[Unmodular]`**, named above so it is not renamed, added when a parameter needs it.
- **Pushing from `#[Scoped]`.**
- **Logging.** The channel attribute is a worked example, not a design.
- **Thread safety**, which the open question below covers.

## Alternatives considered

**A single current scope rather than a stack.** A scoped binding can depend on another module's scoped binding, so
the value has to nest.

**A flag marking a binding scope-aware.** It would be true exactly when an owner exists, and so would say nothing
that the owner does not.

**Looking the scope up when a value is read**, rather than capturing it when the value is built. A nested resolution
moves the stack in between, so the answer read later is not the answer that was true when the value was created.

**Inheriting a scope from the value that caused it.** A `Dependency` stamped `X` resolving to a binding owned by `Y`
must produce `Y`, which is precisely the case inheritance would get wrong.

**Pushing a scope from `#[Scoped]`.** A parameter attribute that silently rescopes an arbitrarily deep subtree is
hard to reason about. It can be added later if a case needs it, and could not be taken back.

**Erroring on an empty stack.** Services disagree about whether an empty stack is a problem, so the decision belongs
to each service rather than to the container.

No other alternatives were weighed.

## Backwards compatibility

- `Dependency` takes one more constructor argument, so everything that builds one supplies the scope.
- `Invocation` carries a scope the container stamps, which its static factories do not supply.

Nothing else changes. `Resolver::resolve()` is untouched, and no binding needs editing: a binding with no owner
behaves exactly as it does today.

## Open questions

- **Whether queue workers run as an in-process thread pool.** A scope stack on a shared container is mutable state
  across concurrent resolutions, and interleaved pushes and pops would corrupt it. A container per thread makes the
  question moot; a shared container needs the stack to be thread-local. Nothing in the record says workers are
  threaded, and the only concurrency anywhere in it is concurrent fragment requests and concurrent index creation.
  It belongs to the bootstrap design, where the driver contract lives.

## Changelog

## Sources

- Issue [#70], Engine - Container - Module scopes, 2026-09-03, edited three minutes later the same day: that there
  is no ambient current module and why, the binding being the identity carrier, the scope stack and why it is a
  stack, `try`/`finally` and the dirty stack it prevents, emptying at the cycle boundary rather than the request
  boundary and the leak it avoids, ownership as the whole signal with no opt-in flag and no gate on reading, the
  empty stack belonging to the service with the logger and schema builder as the two cases, the debug warning,
  capture at creation with the two values acquiring it differently and neither inheriting, consumption through
  factories and resolvers with no signature changed, scope and attributes being orthogonal with the channel table,
  `#[Scoped]` with its two constraints and why it pushes nothing, `Unmodular` and the four names it was chosen over,
  the push sites and the accepted residue, the three rules, the memoisation conflict, every boundary, and the
  thread-pool question.
- Issue [#39], Engine - Modules - Collectors, 2026-06-02, with no edits: `Unscoped` already meaning something
  different for a `#[Collect]` method, which is why the opposite case needs another name.
- Issue [#47], Engine - Container - Circular dependency detection, 2026-08-08, with no edits: popping only on the
  success path leaving stale entries after any failure, which is the precedent for wrapping push and pop.
- Issue [#67], Engine - Sessions, 2026-08-22, and issue [#59], Engine - Database - Schema: tables, indexes and DDL
  objects, 2026-08-19: concurrent fragment requests and concurrent index creation, the only concurrency anywhere in
  the record, which is why the thread-pool question is open rather than answered.

[#39]: https://github.com/thegamepanel/panel/issues/39
[#47]: https://github.com/thegamepanel/panel/issues/47
[#59]: https://github.com/thegamepanel/panel/issues/59
[#67]: https://github.com/thegamepanel/panel/issues/67
[#70]: https://github.com/thegamepanel/panel/issues/70
