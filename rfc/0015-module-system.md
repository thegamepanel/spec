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

I will give the [engine](../GLOSSARY.md#engine) its extension mechanism, in which a [module](../GLOSSARY.md#module) is
either bundled inside the binary or installed as a Composer package and is described by the same
[manifest](../GLOSSARY.md#manifest) and [registrar](../GLOSSARY.md#registrar) metadata either way, compiled ahead of
time in production and reflected at boot in debug. Enabled modules will register and then boot, and every
[binding](../GLOSSARY.md#binding) they register will be assembled into the container's
[catalogue](../GLOSSARY.md#catalogue)s. Everything else a module contributes will be pulled by the component that wants
it, at the moment it wants it, rather than pushed by modules during boot.

## Motivation

Every core feature of the panel is a [module](../GLOSSARY.md#module), as
[ADR-0013](../adr/0013-core-modules.md) decides, so the panel cannot run without a module system. It is not
an optional extension point to be added once the [engine](../GLOSSARY.md#engine) works.

Several designs already stop at it. The event dispatcher in [RFC-0009](0009-events.md) leaves reading listener
attributes off a class to the module system. The HTTP transport in [RFC-0010](0010-http-transport.md) leaves modules
contributing middleware or routes to it. The views in [RFC-0011](0011-views.md) leave collecting template sources and
slot definitions to it. The dependency injection container in [RFC-0001](0001-dependency-injection-container.md) leaves
assembling its [catalogue](../GLOSSARY.md#catalogue)s to it.

Discovering modules also has to be cheap. The panel runs as a worker, as
[ADR-0009](../adr/0009-frankenphp-worker.md) decides, so discovery happens once
at boot and is paid back over every request the worker serves. A developer editing a module still needs to see the
change without a rebuild.

## Proposal

### Where a module comes from

A [module](../GLOSSARY.md#module) will be either a [bundled module](../GLOSSARY.md#bundled-module) or an
[external module](../GLOSSARY.md#external-module), and nothing after discovery will be able to tell which:

| | Bundled | External |
|---|---|---|
| Ships | Inside the binary, in the panel's own codebase and autoloader | In the modules directory, with its own `vendor/` |
| Metadata declared | In code | Under `extra.tgp` in the package's `composer.json` |
| Discovered from | The panel itself | The Composer lock file in the modules directory |
| Core flag | Set | Unset |

An external module will be a Composer package of type `tgp-module`, installed into the modules directory. Its
[module identifier](../GLOSSARY.md#module-identifier) will be taken from its package name, where a bundled module will
declare its own.

External modules will be installed with Composer driven as a library, as
[ADR-0012](../adr/0012-composer-installs.md) decides. That decision also settles the
separate Composer project, the `provide` block generated from the panel's own lock file, and `extra.tgp` as the place
for metadata that Composer's schema does not carry.

Both types will produce the same `ModuleManifest` and `ModuleRegistrar`. The registry will hold one set of them and will
never ask where a module came from, so a first-party feature shipped as a bundled module will be an ordinary module in
every other respect, as [ADR-0013](../adr/0013-core-modules.md) decides.

### What describes a module

A [module](../GLOSSARY.md#module)'s `ModuleManifest` will be an immutable record of its resolved metadata: its
[module identifier](../GLOSSARY.md#module-identifier), vendor, version, name and description, the
[capability](../GLOSSARY.md#capability) declarations it makes, its core flag, the class name of its
[registrar](../GLOSSARY.md#registrar), its icon and its definition. For an
[external module](../GLOSSARY.md#external-module), `ModuleManifestBuilder` will build it from the Composer package,
reading the standard Composer fields from the package itself and everything else from `extra.tgp`. A
[bundled module](../GLOSSARY.md#bundled-module) will construct its manifest directly.

The registrar will be a module's entry class. `ModuleRegistrar` will be an immutable record of what reflection found on
that class, built by `ModuleRegistrarBuilder`: which method carries `#[Register]`, which carries `#[Boot]`, and, for
each method carrying `#[Collect]`, the [collector](../GLOSSARY.md#collector) type it accepts, whether it also carries
`#[Unscoped]`, and whether it takes a `PanelContext` parameter. It will describe the class and never hold an instance of
it.

Both records will be hydrated from an array, so that they survive being written out and read back.

Each will be injectable by identifier through a [resolvable attribute](../GLOSSARY.md#resolvable-attribute), `Manifest`
or `Registrar`, as [ADR-0002](../adr/0002-parameter-attributes.md) decides:

```php
public function __construct(
    #[Manifest('backups')] private ModuleManifest $manifest,
) {}
```

#### Capabilities

A [module](../GLOSSARY.md#module) will declare each [capability](../GLOSSARY.md#capability) in its
[manifest](../GLOSSARY.md#manifest), from a `Capability` enum of those recognised. The record names `CrossRoutes`,
`ExtendSchema`, `ModifyUi` and `DaemonAccess`, and the set will grow with the components that enforce them.

Whatever a capability governs will check it at runtime, and the module system never will. Declaring a capability will
not be the same as being granted it: routing will decide what `CrossRoutes` permits, and the module system will only
carry the declaration.

### Where the metadata comes from

Reflection is the cost this design manages. Every [module source](../GLOSSARY.md#module-source) will implement one
interface:

```php
interface ModuleSource
{
    public function manifests(): array;

    public function registrars(): array;
}
```

There will be two. In production, `CachedModuleSource` will read bundled metadata from a file built into the binary and
external metadata from a file written during discovery, and will reflect nothing. In debug, `LiveModuleSource` will
reflect every [module](../GLOSSARY.md#module) at boot, bundled modules from the panel's own codebase and external
modules from the modules directory. Whatever boots the panel will choose one and hand it to the registry, which will not
know which it was given.

`LiveModuleSource` will exist for developing modules: a change to a module's code will show on the next request, with no
rebuild and no discovery run.

The metadata written for `CachedModuleSource` will go under the `compiled` [root](../GLOSSARY.md#root) from
[RFC-0008](0008-filesystems.md), as PHP the [engine](../GLOSSARY.md#engine) writes for PHP to include, so that it is
opcached rather than decoded on every cold boot. It will be [compiled output](../GLOSSARY.md#compiled-output), not
cached content, and will not go through a filesystem.

Bundled [registrar](../GLOSSARY.md#registrar) metadata will be reflected during the build and written into the binary.
No build tooling exists yet, so nothing produces that file, and `LiveModuleSource` will be the only source a developer
can run.

A module's classes will be able to carry attributes from libraries the panel knows nothing about. Reflection will
therefore match attributes by name and instantiate none of them, as
[ADR-0014](../adr/0014-attribute-presence-flags.md) decides, so a third-party attribute's
constructor will never run because a module was discovered.

### The registry

`ModuleRegistry` will be immutable, built from a [module source](../GLOSSARY.md#module-source) when the panel boots. It
will look up one [module](../GLOSSARY.md#module)'s [manifest](../GLOSSARY.md#manifest) or
[registrar](../GLOSSARY.md#registrar) metadata by [module identifier](../GLOSSARY.md#module-identifier), list every
manifest or every identifier, and say whether a module is enabled, disabled, or known at all.

Which modules are enabled will come from `ModulesEnabled`, the [core configuration](../GLOSSARY.md#core-configuration)
object, read through the configuration [registry](../GLOSSARY.md#registry) between its two seals, as
[RFC-0004](0004-toml-configuration-loading.md) describes. The module system will never scan a directory to work out what
is enabled, and will never repeat the rule that a module's configuration file being present is what enables it.
Configuration will be the one source of that answer.

A module present in the source but missing from the enabled list will be known and disabled. A module in the enabled
list with no manifest will be logged and skipped, because an installation whose configuration still names a module that
has since been removed should still boot.

The registry's own [lifetime](../GLOSSARY.md#lifetime) will follow the mode that produced it, as
[RFC-0007](0007-binding-lifetimes.md) describes. In production it will be `Process`: built once at worker boot from
compiled metadata and shared by every request. In debug it will be `Cycle`: built again for each
[cycle](../GLOSSARY.md#cycle), so that edited module code takes effect at once. The class will be the same in both, and
only what constructs it will differ.

### Register and boot

Every enabled [module](../GLOSSARY.md#module) will pass through two phases, in order:

1. **Register.** Each module's [registrar](../GLOSSARY.md#registrar) will be instantiated, and its `#[Register]` method
   called with an `EngineBuilder`, through which the module will register its [binding](../GLOSSARY.md#binding)s and
   [resolver](../GLOSSARY.md#resolver)s.
2. **Boot.** Once every module has registered and the container has been built, each module's `#[Boot]` method will be
   called with no arguments.

Every module will register before any module boots, so a module's boot may rely on another module having registered. A
module with no method for a phase will be skipped for that phase, without error.

Registrars will be instantiated directly during register, because the container will not exist yet: building it will be
what the phase produces. Boot will call the same registrar objects that register created, by which point the container
will exist.

#### The builder

A [module](../GLOSSARY.md#module) will register its [binding](../GLOSSARY.md#binding)s and
[resolver](../GLOSSARY.md#resolver)s through one surface, `EngineBuilder`, which will hide the container's two
registries, the mutable halves that [ADR-0003](../adr/0003-sealed-catalogues.md)
seals into catalogues. Registering a binding will hand back the builder from
[RFC-0001](0001-dependency-injection-container.md), so a module will describe a binding exactly as the engine does:

```php
#[Register]
public function register(EngineBuilder $engine): void
{
    $engine->bind(BackupStore::class)->to(DatabaseBackupStore::class);
}
```

Registering a resolver will name the [resolvable attribute](../GLOSSARY.md#resolvable-attribute) it answers and the
resolver class, and may make it that attribute's default resolver.

For the length of one module's register call, the builder will be scoped to that module, so every binding registered
through it will record the [module identifier](../GLOSSARY.md#module-identifier) that registered it. Recording the
[owner](../GLOSSARY.md#owner) is this design's part; what an owner means at [resolution](../GLOSSARY.md#resolution) is
left to [RFC-0016](0016-module-scopes.md).

When the register phase ends, the builder will be consumed into the [catalogue](../GLOSSARY.md#catalogue)s the container
is constructed with, `BindingCatalogue` and `ResolverCatalogue`, and will be unavailable afterwards. This will be the
assembly that [RFC-0001](0001-dependency-injection-container.md) leaves to the module lifecycle.

#### Alias chains

An [alias](../GLOSSARY.md#alias) may name another alias. Assembly will flatten every chain, so that each alias names the
[abstract](../GLOSSARY.md#abstract) that owns the [binding](../GLOSSARY.md#binding) directly and
[resolution](../GLOSSARY.md#resolution) stays a single hop.

The chain will be flattened at assembly because a [catalogue](../GLOSSARY.md#catalogue) will be assembled once and never
change, while an alias will be normalised before the [instance cache](../GLOSSARY.md#instance-cache) is consulted on
every resolution, as [RFC-0006](0006-container-improvements.md) describes. Walking the chain at assembly will pay for it
once, at boot, rather than on every resolution of a structure that cannot change.

A chain that comes back to an alias already seen will not be flattened. It will fail the assembly, naming the aliases in
the loop, so that a loop will be a boot failure saying what is wrong rather than a resolution that exhausts the stack.

### Pulling contributions

Everything a [module](../GLOSSARY.md#module) contributes beyond its [binding](../GLOSSARY.md#binding)s will be pulled
rather than pushed. A component will ask for contributions when it wants them, which may be at boot, on first use, or
never.

Each component will define its own [collector](../GLOSSARY.md#collector) type, and a handler that will drive
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

The registry will drive the flow, and the component will decide when it runs. The component will pass its handler to
`ModuleRegistry::collect(CollectorHandler $handler, ?PanelContext $context)`, and the registry will walk every enabled
module. For each, it will look in the [registrar](../GLOSSARY.md#registrar) metadata for a `#[Collect]` method accepting
the handler's collector type, and will skip the module when there is none. Otherwise `create()` will produce a fresh
collector for that module, the module's method will fill it, and `process()` will receive it filled. Once every module
has been walked, `finalise()` will run once.

Each module will receive a fresh collector, so nothing one module contributes can reach or overwrite what another
contributed. The handler will be the only thing that sees them all. A module contributing routes to a backups page will
look like this:

```php
#[Collect]
public function routes(RouteCollector $routes): void
{
    $routes->get('/backups', ListBackups::class);
}
```

The collector type will be taken from the method's type hint. A `#[Collect]` method may also declare a `PanelContext`
parameter, in which case it will take part only when collection runs for that
[panel context](../GLOSSARY.md#panel-context); a method without one will take part whatever the context.

#### Where a contribution reaches

By default, a [module](../GLOSSARY.md#module)'s `#[Collect]` method will contribute within its own module's boundary,
and the `$scoped` flag passed to the handler will say so. `#[Unscoped]` will mark a method as contributing outside that
boundary.

Whether that is allowed will be the handler's decision, never the module system's. The handler will receive both the
flag and the module's [manifest](../GLOSSARY.md#manifest), and so will have the declaration and every
[capability](../GLOSSARY.md#capability) the module declares to decide with. Permissions, for example, will refuse an
unscoped contribution outright, while routing will be able to accept one from a module declaring `CrossRoutes`.

`#[Unscoped]` here will mean outside the module's own contribution boundary. It will be unrelated to the container's
[module scope](../GLOSSARY.md#module-scope)s in [RFC-0016](0016-module-scopes.md), which will be a separate axis.

#### Caching what was collected

There will be no cacheable [collector](../GLOSSARY.md#collector) abstraction, because components will cache differently.
A route collector will be able to cache a form that skips [collection](../GLOSSARY.md#collection) entirely on later
requests, while another collector will gain nothing from caching at all. Each component will cache around collection
however suits it.

### Panel context

`PanelContext` will be an enum with one case for each [panel context](../GLOSSARY.md#panel-context): `Account`, `Server`
and `Platform`. It will scope [collection](../GLOSSARY.md#collection), and routing will use it to decide a route's URL
prefix. For routing, `Server` will be a subcontext of `Account`; for collection, it will be a value in its own right.

It will be defined among the [engine](../GLOSSARY.md#engine)'s shared values, alongside the client address value object,
rather than inside collection. Collection is where it was first written down, but routing will need it too, which is the
promotion [RFC-0010](0010-http-transport.md) records as unsettled. With two consumers it will be shared, and neither
will own it.

### Errors

Every exception the module system throws will implement `ModuleException`, a marker contract. `ModuleSourceException`
will be thrown when metadata is missing or unreadable, with a message saying what to do: run discovery, or check the
installation. `UnknownModuleException` will be thrown when a [manifest](../GLOSSARY.md#manifest) or
[registrar](../GLOSSARY.md#registrar) is asked for by an identifier that is not known.

An enabled [module](../GLOSSARY.md#module) with no manifest will be a warning rather than an exception, and the module
will be skipped.

### Out of scope

- **Module scopes.** What the [owner](../GLOSSARY.md#owner) recorded on a [binding](../GLOSSARY.md#binding) means at
  [resolution](../GLOSSARY.md#resolution), and what a [module](../GLOSSARY.md#module) is handed because of it, is its
  own design, [RFC-0016](0016-module-scopes.md).
- **Module resources.** Provisioning a module's schema, directories or filesystems.
- **The build step.** Reflecting bundled [registrar](../GLOSSARY.md#registrar)s during the build and writing them into
  the binary will belong to bootstrapping, along with everything else that will put the [engine](../GLOSSARY.md#engine)
  together.
- **Enabling and disabling modules.** That will belong to the command line, as
  [RFC-0004](0004-toml-configuration-loading.md) describes.
- **Which collectors exist.** Each component will define its own [collector](../GLOSSARY.md#collector) and handler. This
  design will define the mechanism and nothing that uses it.
- **Enforcing capabilities.** That will belong to whatever a [capability](../GLOSSARY.md#capability) governs.
- **A module API package and an SDK** for third-party development.
- **Dependencies between modules.** Nothing here will resolve a dependency between two modules or order modules by one.

## Alternatives considered

This design rests on five decisions recorded separately:
[ADR-0002](../adr/0002-parameter-attributes.md), dependencies selecting their
instance through parameter attributes;
[ADR-0003](../adr/0003-sealed-catalogues.md), mutable registries sealed into
immutable catalogues; [ADR-0009](../adr/0009-frankenphp-worker.md), the panel
running as a FrankenPHP worker in a single binary;
[ADR-0012](../adr/0012-composer-installs.md), modules installed with Composer driven
as a library; and [ADR-0013](../adr/0013-core-modules.md), core features being modules.

**Collection as a third lifecycle phase**, with modules pushing every contribution during boot. I rejected this because
a component would then receive contributions whether or not it is ever used, and would have nowhere to decide when to
collect. Pulling lets a component collect on first use, or never.

**A cacheable collector abstraction.** I rejected this because components cache collected data in different shapes, and
some gain nothing from caching, so one abstraction would have to cover cases that do not resemble each other.

**Following alias chains at runtime**, walking from one alias to the next on each lookup. I rejected this because the
walk would repeat on every [resolution](../GLOSSARY.md#resolution), on the path taken before the
[instance cache](../GLOSSARY.md#instance-cache) is consulted, for a structure fixed at boot. A loop would also be found
by exhausting the stack rather than when the [catalogue](../GLOSSARY.md#catalogue) was assembled.

## Backwards compatibility

Nothing will break. No module exists, and the modules directory in the repository is empty.

The container is constructed with its [catalogue](../GLOSSARY.md#catalogue)s today, and will gain the step that builds
them. That step is what [RFC-0001](0001-dependency-injection-container.md) left to this design, not a change to it.

## Open questions

## Changelog

- 2026-09-16: Alias chains are flattened when the catalogue is assembled, which settles the only open question.
- 2026-10-05: Reworded in the readable house style, and the project's own vocabulary linked to the glossary. The
  design is unchanged.
- 2026-11-18: Rewrote the document in full, with no change to the design.

## Sources

- Issue [#35], Engine - Modules, original text of 2026-06-01 in its edit history: the module system as the extension
  mechanism, the two sources and how they differ, both producing the same structures with the registry not telling them
  apart, capabilities declared in manifests and enforced elsewhere, the two lifecycle phases driven by a bootstrapper,
  collection being pull-based and not part of the lifecycle, and the module API package left out.
- Issue [#35], revised on 2026-06-02 in its edit history: the `ModuleSource` seam with its compiled and reflecting
  implementations, bootstrap selecting one, and the registry not knowing which it was given. A later edit on 2026-09-13
  replaced the subissue names with issue links and changed nothing else.
- Issue [#36], Engine - Modules - Discovery and Manifest, 2026-06-02, with no edits: both source implementations and
  what each reads, external discovery from the Composer lock file, the metadata declared under `extra.tgp`, the manifest
  and registrar and their builders, hydration from an array, the capability enum with the four it names, the four
  registrar attributes, the prebuilt bundled file, and the `#[Manifest]` and `#[Registrar]` attributes with their
  resolvers. It places external metadata in a JSON file in the cache directory; issue [#73] supersedes that.
- Issue [#37], Engine - Modules - Registry, 2026-06-02, with no edits: the immutable registry built from a source, every
  lookup, enabled modules coming from configuration, a disabled module still being known, an enabled module with no
  manifest being logged and skipped, an unknown identifier throwing, a missing source being an actionable error, the
  collection entry point, and the registry being built once per worker in production and per request in debug.
- Issue [#38], Engine - Modules - Lifecycle, 2026-06-02, with no edits: the two phases and their order, every module
  registering before any boots, `EngineBuilder` with its two methods and its scoping to a module during each
  registration call, its consumption into the immutable container, registrars instantiated directly during register
  because the container does not exist, and a missing method for a phase being skipped silently.
- Issue [#39], Engine - Modules - Collectors, 2026-06-02, with no edits: collection being pull-based and triggered by
  the component, the collector marker and the handler contract with its four methods, the seven steps of the flow, a
  fresh collector per module, the collector type inferred from the type hint, the optional `PanelContext` parameter
  filtering participation and a method without it participating whatever the context, `#[Unscoped]` with the decision
  left to the handler, permissions and routes as the two worked examples, `PanelContext` and its three cases with
  `Server` as a subcontext of `Account` in routing, and caching left to each component with no cacheable collector
  abstraction.
- Issue [#44], Engine - Container - Alias cache key mismatch, text of 2026-08-08 in its edit history: chained aliases
  not resolving, and the choice between flattening them when catalogues are built and following them at runtime being
  left to this design.
- Issue [#70], Engine - Container - Module scopes, 2026-09-03, edited the same day: a binding registered through a
  module-scoped builder knowing its owner, which is what module scopes are built on, and `Unscoped` already meaning
  something different for a `#[Collect]` method.
- Issue [#73], Engine - Filesystems - Paths: compiled, 2026-09-03, edited on 2026-09-13: external module metadata moving
  out of the cache directory to the compiled root, as an includable PHP file opcached rather than decoded on every cold
  boot.
- Issue [#43], Engine - Container - Memoise class-level attribute lookups, original text of 2026-08-08 in its edit
  history: third-party classes carrying attributes from unrelated libraries once the module system lands, which is why
  matching is by name and nothing is instantiated.
- `PanelContext` being defined in the engine's shared values rather than inside collection; the registry's two
  construction modes expressed as process and cycle lifetimes; `EngineBuilder` being the module-facing surface over the
  container's two registries; alias chains being flattened at assembly, with a cycle failing the assembly and naming the
  aliases in it; and the `ModuleException` marker with `ModuleSourceException` and `UnknownModuleException`: first
  written down on 2026-09-16.

[#35]: https://github.com/thegamepanel/panel/issues/35
[#36]: https://github.com/thegamepanel/panel/issues/36
[#37]: https://github.com/thegamepanel/panel/issues/37
[#38]: https://github.com/thegamepanel/panel/issues/38
[#39]: https://github.com/thegamepanel/panel/issues/39
[#43]: https://github.com/thegamepanel/panel/issues/43
[#44]: https://github.com/thegamepanel/panel/issues/44
[#70]: https://github.com/thegamepanel/panel/issues/70
[#73]: https://github.com/thegamepanel/panel/issues/73
