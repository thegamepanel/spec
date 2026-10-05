---
id: RFC-0015
title: Module system
status: proposed
created: 2026-09-16
decided:
depends: [ADR-0002, ADR-0003, ADR-0009, ADR-0012, ADR-0013, RFC-0001, RFC-0004, RFC-0007, RFC-0008]
updates: []
obsoletes: []
---

# RFC-0015: Module system

## Abstract

We will add the [engine](../GLOSSARY.md#engine)'s extension mechanism, in which a [module](../GLOSSARY.md#module) is
either bundled inside the binary or installed as a Composer package, and is described by the same
[manifest](../GLOSSARY.md#manifest) and [registrar](../GLOSSARY.md#registrar) metadata either way. Metadata will be
compiled ahead of time in production and reflected at boot in debug, behind one interface the registry consumes. Enabled
modules will pass through two phases, register and boot, and every [binding](../GLOSSARY.md#binding) they register will
be assembled into the container's [catalogue](../GLOSSARY.md#catalogue)s. Everything else a module contributes will be
pulled: a component will ask every module for contributions at the moment it needs them, rather than modules pushing
during boot.

## Motivation

Each core feature of the panel is a [module](../GLOSSARY.md#module), as
[ADR-0013](../adr/0013-core-features-are-modules.md) decides, so the panel cannot run without a module system. It is
not an optional extension point added once the [engine](../GLOSSARY.md#engine) works.

Several designs are already written against it and cannot be finished without it. The event dispatcher in
[RFC-0009](0009-events.md) leaves reading listener attributes off a class to the module system. The HTTP transport in
[RFC-0010](0010-http-transport.md) leaves modules contributing middleware or routes to it. The views in
[RFC-0011](0011-views.md) leave collecting template sources and slot definitions to it. The dependency injection
container in [RFC-0001](0001-dependency-injection-container.md) leaves assembling its catalogues to it.

Modules also have to be discovered cheaply. The panel runs as a worker, as
[ADR-0009](../adr/0009-the-panel-runs-as-a-frankenphp-worker-in-a-single-binary.md) decides, so discovery happens
once at boot and is paid back over every request the worker serves. A developer editing a module still needs their
change to be visible without a rebuild.

## Proposal

### Module sources

A [module](../GLOSSARY.md#module) will be either a [bundled module](../GLOSSARY.md#bundled-module) or an
[external module](../GLOSSARY.md#external-module), and nothing downstream of discovery can tell which:

| | Bundled | External |
|---|---|---|
| Ships | Inside the binary, in the panel's own codebase and autoloader | In the modules directory, with its own `vendor/` |
| Metadata declared | In code | In `extra.tgp` in the package's `composer.json` |
| Discovered from | The panel itself | The Composer lock file in the modules directory |
| Core flag | Set | Unset |

An external module will be a Composer package of type `tgp-module`, installed into the modules directory. Its
[module identifier](../GLOSSARY.md#module-identifier) will be derived from its package name, and a bundled module will
declare its identifier.

External modules will be installed with Composer driven as a library, as
[ADR-0012](../adr/0012-modules-are-installed-with-composer-used-as-a-library.md) decides. That decision also settles
the separate Composer project, the `provide` block generated from the panel's own lock file, and `extra.tgp` as the
place for metadata that Composer's schema does not carry.

Both types of module will produce the same `ModuleManifest` and `ModuleRegistrar`. The registry will hold one set of
them and never ask where a module came from. First-party features will therefore ship as bundled modules and be ordinary
modules in every other respect, as [ADR-0013](../adr/0013-core-features-are-modules.md) decides.

### Metadata

`ModuleManifest` will be immutable and hold one module's resolved metadata: its
[module identifier](../GLOSSARY.md#module-identifier), vendor, version, name, description, declared capabilities, core
flag, registrar class name, icon and definition. `ModuleManifestBuilder` will build it for an
[external module](../GLOSSARY.md#external-module) from the Composer package, reading the standard Composer fields from
the package itself and the rest from `extra.tgp`. A [bundled module](../GLOSSARY.md#bundled-module) will construct its
manifest directly.

A module's [registrar](../GLOSSARY.md#registrar) is its entry class. `ModuleRegistrar` will be immutable and hold what
reflection found on that class, and `ModuleRegistrarBuilder` will build it. It will record which method carries
`#[Register]`, which carries `#[Boot]`, and, for each `#[Collect]` method, the [collector](../GLOSSARY.md#collector)
type it accepts, whether it carries `#[Unscoped]`, and whether it takes a `PanelContext` parameter. It will hold
metadata about the class, never an instance of it.

Both will be hydrated from an array, so both survive being written out and read back.

Each will be injectable by identifier through a [resolvable attribute](../GLOSSARY.md#resolvable-attribute), `Manifest`
or `Registrar`, as [ADR-0002](../adr/0002-dependencies-select-their-instance-through-parameter-attributes.md) decides:

```php
public function __construct(
    #[Manifest('backups')] private ModuleManifest $manifest,
) {}
```

#### Capabilities

A module will declare each [capability](../GLOSSARY.md#capability) in its [manifest](../GLOSSARY.md#manifest), and
`Capability` will be the enum of those recognised. The record names `CrossRoutes`, `ExtendSchema`, `ModifyUi` and
`DaemonAccess`; the set grows with the components that enforce them.

Whatever enforces a capability will check it at runtime, never the module system. Declaring a capability is not being
granted it: routing decides what `CrossRoutes` permits, and the module system only carries the declaration.

### Sources of metadata

Reflection is the cost this design manages. Every [module source](../GLOSSARY.md#module-source) will implement one
interface:

```php
interface ModuleSource
{
    public function manifests(): array;

    public function registrars(): array;
}
```

| Implementation | Used in | Behaviour |
|---|---|---|
| `CachedModuleSource` | Production | Reads bundled metadata from a file built into the binary, and external metadata from a file written during discovery. Reflects nothing. |
| `LiveModuleSource` | Debug | Reflects every module at boot, bundled from the panel's own codebase and external from the modules directory. |

Whatever boots the panel will choose the implementation and hand it to the registry, which will not know which it was
given.

`LiveModuleSource` will exist for module development: a change to module code will be visible on the next request, with
no rebuild and no discovery run.

Module metadata written for `CachedModuleSource` will go to the `compiled` [root](../GLOSSARY.md#root) from
[RFC-0008](0008-filesystems.md), as PHP the [engine](../GLOSSARY.md#engine) writes for PHP to include, so it will be
opcached rather than decoded on every cold boot. It will not be cached content and will not go through a filesystem.

Bundled [registrar](../GLOSSARY.md#registrar) metadata will be reflected during the build and written into the binary.
No build tooling exists yet, so nothing produces that file today, and `LiveModuleSource` will be the only implementation
a developer can run.

A module class may carry attributes from libraries the panel knows nothing about. Reflection will therefore match
attributes by name and instantiate none of them, as
[ADR-0014](../adr/0014-class-level-attributes-are-memoised-as-presence-flags.md) decides, so a third-party attribute's
constructor will never run because a module was discovered.

### The registry

`ModuleRegistry` will be immutable, built from a [module source](../GLOSSARY.md#module-source) when the panel boots.

| Lookup | Returns |
|---|---|
| `manifest(string $ident)` | One module's manifest. |
| `registrar(string $ident)` | One module's registrar metadata. |
| `manifests()` | Every manifest. |
| `idents()` | Every identifier. |
| `isEnabled()`, `isDisabled()`, `has()` | Whether a module is enabled, disabled, or known at all. |

Which modules are enabled will come from `ModulesEnabled`, the [core configuration](../GLOSSARY.md#core-configuration)
object, read through the configuration [registry](../GLOSSARY.md#registry) between its two seals, as
[RFC-0004](0004-toml-configuration-loading.md) describes. The module system will never scan a directory to work out what
is enabled, and will never repeat the rule that a module's configuration file being present is what enables it.
Configuration is the single source of that.

A module present in the source but not in the enabled list will be known and disabled. A module in the enabled list with
no [manifest](../GLOSSARY.md#manifest) will be logged and skipped, because an installation whose configuration names a
module that is no longer installed should still boot.

The registry's own [lifetime](../GLOSSARY.md#lifetime) will follow the mode that produced it, as
[RFC-0007](0007-binding-lifetimes.md) describes:

| Mode | Lifetime | Effect |
|---|---|---|
| Production | `Process` | Built once at worker boot from compiled metadata, and shared by every request. |
| Debug | `Cycle` | Rebuilt for each [cycle](../GLOSSARY.md#cycle), so edited module code takes effect immediately. |

The class will be the same in both. Only what constructs it will differ.

### Lifecycle

Every enabled module will pass through two phases, in order:

1. **Register.** Each module's [registrar](../GLOSSARY.md#registrar) will be instantiated, and its `#[Register]` method
   called with an `EngineBuilder`, through which the module registers its bindings and resolvers.
2. **Boot.** Once every module has registered and the container exists, each module's `#[Boot]` method will be called
   with no arguments.

Every module will register before any module boots, so a module's boot may depend on another module having registered. A
module with no method for a phase will be skipped for that phase, without error.

Registrars will be instantiated directly during register, because the container does not exist yet: building it is what
the phase produces. By boot the container will exist, and the registrars instantiated during register will be the same
objects.

#### The builder

`EngineBuilder` will be the mutable half of the container's registries, per
[ADR-0003](../adr/0003-mutable-registries-are-sealed-into-immutable-catalogues.md), presented to a module as one
surface:

| Method | Effect |
|---|---|
| `bind(string $abstract): BindingBuilder` | Registers a [binding](../GLOSSARY.md#binding), returning the builder from [RFC-0001](0001-dependency-injection-container.md). |
| `resolver(string $resolvable, string $resolver, bool $default = false)` | Registers a [resolver](../GLOSSARY.md#resolver) against a [resolvable attribute](../GLOSSARY.md#resolvable-attribute). |

A builder will be scoped to a module for the duration of that module's register call, so every binding registered
through it will record the [module identifier](../GLOSSARY.md#module-identifier) that registered it. Recording the owner
is this design's part. What an owner means at [resolution](../GLOSSARY.md#resolution) is left to
[RFC-0016](0016-module-scopes.md).

After the register phase the builder will be consumed, producing each [catalogue](../GLOSSARY.md#catalogue) the
container is constructed with, `BindingCatalogue` and `ResolverCatalogue`, and will be unavailable afterwards. This is
the assembly [RFC-0001](0001-dependency-injection-container.md) leaves to the module lifecycle.

#### Alias chains

An [alias](../GLOSSARY.md#alias) may name another alias. Assembly will flatten every chain, so each alias names the
[abstract](../GLOSSARY.md#abstract) that owns the [binding](../GLOSSARY.md#binding) directly and
[resolution](../GLOSSARY.md#resolution) stays a single hop.

Flattening will happen at assembly because a [catalogue](../GLOSSARY.md#catalogue) is assembled once and never changes
afterwards, while an alias is normalised before the instance cache is consulted on every resolution, as
[RFC-0006](0006-container-improvements.md) describes. Flattening at assembly will pay for the walk once, at boot, rather
than on every resolution for a structure that cannot change.

A chain that returns to an alias already seen cannot be flattened. It will fail the assembly, naming the aliases in
the loop. A loop will therefore be a boot failure that says what is wrong, rather than a resolution that exhausts the
stack.

### Collection

Everything a [module](../GLOSSARY.md#module) contributes beyond bindings will be pulled rather than pushed. A component
will ask for contributions when it wants them, which may be at boot, on first use, or never.

A component will define its own [collector](../GLOSSARY.md#collector) type and a handler that drives
[collection](../GLOSSARY.md#collection) for it:

```php
interface Collector {}

interface CollectorHandler
{
    public function collects(): string;

    public function create(ModuleManifest $manifest, ?PanelContext $context, bool $scoped): Collector;

    public function process(Collector $collector, ModuleManifest $manifest, ?PanelContext $context, bool $scoped): void;

    public function finalise(): void;
}
```

The registry will drive the flow, and the component will decide when it runs:

1. The component will pass its handler to `ModuleRegistry::collect(CollectorHandler $handler, ?PanelContext $context)`.
2. The registry will walk every enabled module.
3. For each, it will check the [registrar](../GLOSSARY.md#registrar) metadata for a `#[Collect]` method accepting that
   collector type, and skip the module when there is none.
4. `create()` will produce a fresh collector for that module.
5. The module's method will populate it.
6. `process()` will receive the populated collector.
7. After every module, `finalise()` will run once.

A module will receive a fresh collector each time, so nothing a module contributes can reach or overwrite what
another contributed. The handler will be the only thing that sees them all.

```php
#[Collect]
public function routes(RouteCollector $routes): void
{
    $routes->get('/backups', ListBackups::class);
}
```

The collector type will be taken from the method's type hint. A `#[Collect]` method may also declare a `PanelContext`
parameter, in which case it will participate only when collection runs for that
[panel context](../GLOSSARY.md#panel-context).

#### Scope of a contribution

By default a [module](../GLOSSARY.md#module)'s `#[Collect]` method will contribute within its own module's boundary, and
the `$scoped` flag passed to the handler will say so. `#[Unscoped]` will mark a method as contributing outside it.

Whether that is permitted will be the handler's decision, never the module system's. The handler will receive both the
flag and the module's [manifest](../GLOSSARY.md#manifest), so it has the declaration and every
[capability](../GLOSSARY.md#capability) the module declares to decide with. For example, permissions refuse unscoped
contributions outright, while routing may allow one from a module declaring `CrossRoutes`.

`#[Unscoped]` here means outside the module's own contribution boundary. It is unrelated to the container's module
scopes in [RFC-0016](0016-module-scopes.md), which are a separate axis.

#### Caching what was collected

There will be no cacheable [collector](../GLOSSARY.md#collector) abstraction. Different components cache differently: a
route collector can cache a form that skips [collection](../GLOSSARY.md#collection) entirely on later requests, while
another gains nothing from caching at all. A component will cache around collection however suits it.

### Panel context

`PanelContext` will be an enum with one case for each [panel context](../GLOSSARY.md#panel-context): `Account`, `Server`
and `Platform`. It will be used to scope [collection](../GLOSSARY.md#collection), and routing will use it to decide a
route's URL prefix.

`Server` will be a subcontext of `Account` for routing, and a value in its own right for collection.

It will be defined in the [engine](../GLOSSARY.md#engine)'s shared values rather than inside collection, alongside the
client address value object. Collection is where it was first written down, but routing needs it too, which is the
promotion [RFC-0010](0010-http-transport.md) records as unsettled. Two consumers make it shared, so it will not be owned
by either.

### Errors

Every exception will implement `ModuleException`, a marker contract.

| Exception | Thrown when |
|---|---|
| `ModuleSourceException` | Metadata is missing or unreadable. Its message says what to do: run discovery, or check the installation. |
| `UnknownModuleException` | A manifest or registrar is asked for by an identifier that is not known. |

An enabled [module](../GLOSSARY.md#module) with no [manifest](../GLOSSARY.md#manifest) will be a warning rather than an
exception, and the module will be skipped.

### Out of scope

- **Module scopes.** What the owner recorded on a [binding](../GLOSSARY.md#binding) means at
  [resolution](../GLOSSARY.md#resolution), and what a module is handed because of it, is its own design,
  [RFC-0016](0016-module-scopes.md).
- **Module resources.** Provisioning a [module](../GLOSSARY.md#module)'s schema, directories or filesystems.
- **The build step.** Reflecting bundled registrars during the build and writing them into the binary belongs to
  bootstrapping, along with everything else that composes the [engine](../GLOSSARY.md#engine).
- **Enabling and disabling modules.** That belongs to the command line, as
  [RFC-0004](0004-toml-configuration-loading.md) describes.
- **Which collectors exist.** Each component defines its own [collector](../GLOSSARY.md#collector) and handler. This
  design defines the mechanism and nothing that uses it.
- **Enforcing capabilities.** That belongs to whatever a [capability](../GLOSSARY.md#capability) governs.
- **A module API package and an SDK** for third-party development.
- **Inter-module dependencies.** Nothing here will resolve a dependency between two modules, or order them by one.

## Alternatives considered

The decisions this design rests on are recorded separately, with the alternatives each one rejected:

- dependencies selecting their instance through parameter attributes, in
  [ADR-0002](../adr/0002-dependencies-select-their-instance-through-parameter-attributes.md)
- sealing mutable registries into immutable catalogues, in
  [ADR-0003](../adr/0003-mutable-registries-are-sealed-into-immutable-catalogues.md)
- the panel running as a FrankenPHP worker in a single binary, in
  [ADR-0009](../adr/0009-the-panel-runs-as-a-frankenphp-worker-in-a-single-binary.md)
- installing modules with Composer driven as a library, in
  [ADR-0012](../adr/0012-modules-are-installed-with-composer-used-as-a-library.md)
- core features being modules, in [ADR-0013](../adr/0013-core-features-are-modules.md)

**Collection as a third lifecycle phase**, with modules pushing every contribution during boot. We rejected this
because a component would then receive contributions whether or not it is ever used, and would have nowhere to put the
decision of when to collect. Pulling lets a component collect on first use, or never.

**A cacheable collector abstraction.** We rejected this because components cache collected data in different shapes,
and some gain nothing from caching, so an abstraction would have to cover cases that do not resemble each other.

**Following alias chains at runtime**, walking from one alias to the next on each lookup. We rejected this because the
walk would repeat on every resolution, in the path taken before the instance cache is consulted, for a structure fixed
at boot. A loop would also be found by exhausting the stack rather than when the catalogue was assembled.

No other alternatives were weighed.

## Backwards compatibility

Nothing will break. No module exists, and the modules directory in the repository is empty.

The container is constructed with its catalogues today, and will gain the step that builds them. That step is what
[RFC-0001](0001-dependency-injection-container.md) left to this design, not a change to it.

## Open questions

## Changelog

- 2026-09-16: Alias chains are flattened when the catalogue is assembled, which settles the only open question.
- 2026-10-05: Reworded in the readable house style, and the project's own vocabulary linked to the glossary. The
  design is unchanged.

## Sources

- Issue [#35], Engine - Modules, original text of 2026-06-01 in its edit history: the module system as the extension
  mechanism, the two sources and their differences, both producing the same structures with the registry not
  distinguishing them, capabilities declared in manifests and enforced elsewhere, the two lifecycle phases driven by
  a bootstrapper, collection being pull-based and not part of the lifecycle, and the module API package left out.
- Issue [#35], revised on 2026-06-02 in its edit history: the `ModuleSource` seam with its compiled and reflecting
  implementations, bootstrap selecting one, and the registry not knowing which it was given. A later edit on
  2026-09-13 replaced the subissue names with issue links and changed nothing else.
- Issue [#36], Engine - Modules - Discovery and Manifest, 2026-06-02, with no edits: both source implementations and
  what each reads, external discovery from the Composer lock file, the metadata declared under `extra.tgp`, the
  manifest and registrar and their builders, hydration from an array, the capability enum with the four it names,
  the four registrar attributes, the prebuilt bundled file, and the `#[Manifest]` and `#[Registrar]` attributes with
  their resolvers. It places external metadata in a JSON file in the cache directory; issue [#73] supersedes that.
- Issue [#37], Engine - Modules - Registry, 2026-06-02, with no edits: the immutable registry built from a source,
  every lookup, enabled modules coming from configuration, a disabled module still being known, an enabled module
  with no manifest being logged and skipped, an unknown ident throwing, a missing source being an actionable error,
  the collection entry point, and the registry being built once per worker in production and per request in debug.
- Issue [#38], Engine - Modules - Lifecycle, 2026-06-02, with no edits: the two phases and their order, every module
  registering before any boots, `EngineBuilder` with its two methods and its scoping to a module during each
  registration call, its consumption into the immutable container, registrars instantiated directly during register
  because the container does not exist, and a missing method for a phase being skipped silently.
- Issue [#39], Engine - Modules - Collectors, 2026-06-02, with no edits: collection being pull-based and triggered
  by the component, the collector marker and the handler contract with its four methods, the seven steps of the
  flow, a fresh collector per module, the collector type inferred from the type hint, the optional `PanelContext`
  parameter filtering participation, `#[Unscoped]` with the decision left to the handler, permissions and routes as
  the two worked examples, `PanelContext` and its three cases with `Server` as a subcontext of `Account` in routing,
  and caching left to each component with no cacheable collector abstraction.
- Issue [#44], Engine - Container - Alias cache key mismatch, 2026-08-08, edited the same day: chained aliases not
  resolving, and the choice between flattening them when catalogues are built and following them at runtime being
  left to this design.
- Issue [#70], Engine - Container - Module scopes, 2026-09-03, edited the same day: a binding registered through a
  module-scoped builder knowing its owner, which is what module scopes are built on, and `Unscoped` already meaning
  something different for a `#[Collect]` method.
- Issue [#73], Engine - Filesystems - Paths: compiled, 2026-09-03, edited on 2026-09-13: external module metadata
  moving out of the cache directory to the compiled root, as an includable PHP file opcached rather than decoded on
  every cold boot.
- Issue [#43], Engine - Container - Memoise class-level attribute lookups, 2026-08-08, with no edits: third-party
  classes carrying attributes from unrelated libraries once the module system lands, which is why matching is by
  name and nothing is instantiated.
- `PanelContext` being defined in the engine's shared values rather than inside collection; the registry's two
  construction modes expressed as process and cycle lifetimes; `EngineBuilder` being the module-facing surface over
  the container's two registries; alias chains being flattened at assembly, with a cycle failing the assembly and
  naming the aliases in it; and the `ModuleException` marker with `ModuleSourceException` and
  `UnknownModuleException`: first written down on 2026-09-16.

[#35]: https://github.com/thegamepanel/panel/issues/35
[#36]: https://github.com/thegamepanel/panel/issues/36
[#37]: https://github.com/thegamepanel/panel/issues/37
[#38]: https://github.com/thegamepanel/panel/issues/38
[#39]: https://github.com/thegamepanel/panel/issues/39
[#43]: https://github.com/thegamepanel/panel/issues/43
[#44]: https://github.com/thegamepanel/panel/issues/44
[#70]: https://github.com/thegamepanel/panel/issues/70
[#73]: https://github.com/thegamepanel/panel/issues/73
