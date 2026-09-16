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

The engine's extension mechanism. Modules come from two sources, bundled inside the binary or installed as Composer
packages, and both produce the same manifest and registrar metadata. Metadata is compiled ahead of time in
production and reflected at boot in debug, behind one interface the registry consumes. Enabled modules pass through
two lifecycle phases, register and boot, and the bindings they register are assembled into the container's
catalogues. Everything else a module contributes is pulled: a component asks every module for contributions at the
moment it needs them, rather than modules pushing during boot.

## Motivation

Core features of the panel are modules, per [ADR-0013](../adr/0013-core-features-are-modules.md), so the panel
cannot run without a module system. It is not an optional extension point added once the engine works.

Several components are already written against it and cannot be finished without it. Events leaves reading listener
attributes off a class to the module system, per [RFC-0009](0009-events.md). HTTP leaves modules contributing
middleware or routes to it, per [RFC-0010](0010-http-transport.md). Views leaves collecting template sources and
slot definitions to it, per [RFC-0011](0011-views.md). The container leaves assembling its catalogues to it, per
[RFC-0001](0001-dependency-injection-container.md).

Modules also have to be discovered cheaply. The panel runs as a worker, per
[ADR-0009](../adr/0009-the-panel-runs-as-a-frankenphp-worker-in-a-single-binary.md), so discovery happens once at
boot and is paid back over every request the worker serves, while a developer editing a module needs their change
visible without a rebuild.

## Proposal

### Concepts

| Term | Meaning |
|---|---|
| Module | A unit of functionality registering its own bindings and contributions, bundled or external. |
| Bundled module | A first-party module shipped inside the binary, with its manifest declared in code. |
| External module | A Composer package of type `tgp-module`, installed into the modules directory. |
| Ident | A module's identifier, derived from its package name for an external module and declared for a bundled one. |
| Manifest | The resolved metadata describing one module. |
| Registrar | A module's entry class, and the reflected metadata describing which of its methods do what. |
| Capability | Something a module declares it does, checked by whatever enforces it. |
| Source | Where manifests and registrar metadata come from, compiled or reflected. |
| Collector | What a component hands to modules to contribute to, for one type of contribution. |
| Collection | A component asking every enabled module to contribute, at a moment it chooses. |
| Panel context | The part of the panel something belongs to: an account, a server or the platform. |

### Components

| Component | Responsibility |
|---|---|
| `ModuleManifest` | Immutable metadata for one module. |
| `ModuleRegistrar` | Immutable reflected metadata about a module's registrar class. |
| `ModuleManifestBuilder`, `ModuleRegistrarBuilder` | Build the two from a Composer package and by reflection. |
| `ModuleSource` | Contract providing every manifest and registrar, however they were produced. |
| `CachedModuleSource`, `LiveModuleSource` | The compiled and the reflecting implementations. |
| `ModuleRegistry` | Immutable. Holds the known modules, answers lookups, and drives collection. |
| `EngineBuilder` | Mutable. What a module registers bindings and resolvers through. |
| `Capability` | Enum of the capabilities a module may declare. |
| `Register`, `Boot`, `Collect`, `Unscoped` | Attributes marking what a registrar's methods do. |
| `Manifest`, `Registrar` | Resolvable attributes injecting a module's metadata. |
| `Collector`, `CollectorHandler` | The contracts a component implements to collect from modules. |
| `PanelContext` | Enum naming the part of the panel something belongs to. |
| `ModuleException` | Marker contract implemented by every exception the component throws. |

### Module sources

A module is bundled or external, and nothing downstream of discovery can tell which:

| | Bundled | External |
|---|---|---|
| Ships | Inside the binary, in the panel's own codebase and autoloader | In the modules directory, with its own `vendor/` |
| Metadata declared | In code | In `extra.tgp` in the package's `composer.json` |
| Discovered from | The panel itself | The Composer lock file in the modules directory |
| Core flag | Set | Unset |

External modules are Composer packages installed with Composer driven as a library, per
[ADR-0012](../adr/0012-modules-are-installed-with-composer-used-as-a-library.md), which also settles the separate
Composer project, the `provide` block generated from the panel's own lock file, and `extra.tgp` as where metadata
Composer's schema does not carry lives.

Both produce the same `ModuleManifest` and `ModuleRegistrar`. The registry holds one set and never asks where a
module came from. First-party features therefore ship as bundled modules and are ordinary modules in every other
respect, per [ADR-0013](../adr/0013-core-features-are-modules.md).

### Metadata

`ModuleManifest` is immutable and holds one module's resolved metadata: its ident, vendor, version, name,
description, declared capabilities, core flag, registrar class name, icon and definition. For an external module it
is built from the Composer package, with the standard Composer fields read from the package itself and the rest read
from `extra.tgp`. A bundled module constructs its manifest directly.

`ModuleRegistrar` is immutable and holds what reflection found on the registrar class: which method carries
`#[Register]`, which carries `#[Boot]`, and for each `#[Collect]` method the collector type it accepts, whether it
carries `#[Unscoped]`, and whether it takes a `PanelContext` parameter. It holds metadata about the class, never an
instance of it.

Both are hydrated from an array, so both survive being written out and read back.

Each is injectable by ident through a resolvable attribute, per
[ADR-0002](../adr/0002-dependencies-select-their-instance-through-parameter-attributes.md):

```php
public function __construct(
    #[Manifest('backups')] private ModuleManifest $manifest,
) {}
```

#### Capabilities

A module declares capabilities in its manifest, and `Capability` is the enum of those recognised. The record names
`CrossRoutes`, `ExtendSchema`, `ModifyUi` and `DaemonAccess`; the set grows with the components that enforce them.

A capability is checked at runtime by whatever enforces it, never by the module system. Declaring one is not being
granted it: routing decides what `CrossRoutes` permits, and the module system only carries the declaration.

### Sources of metadata

Reflection is the cost this design manages. `ModuleSource` is the seam:

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

Whatever boots the panel chooses the implementation and hands it to the registry, which does not know which it was
given.

`LiveModuleSource` is what makes module development bearable: a change to module code is visible on the next
request, with no rebuild and no discovery run.

Module metadata written for `CachedModuleSource` goes to the `compiled` root, per [RFC-0008](0008-filesystems.md),
as PHP the engine writes for PHP to include, so it is opcached rather than decoded on every cold boot. It is not
cached content and does not go through a filesystem.

Bundled registrar metadata is reflected during the build and written into the binary. No build tooling exists yet,
so nothing produces that file today, and `LiveModuleSource` is the only implementation a developer can run.

A module class may carry attributes from libraries the panel knows nothing about. Reflection therefore matches
attributes by name and instantiates none of them, per
[ADR-0014](../adr/0014-class-level-attributes-are-memoised-as-presence-flags.md), so a third-party attribute's
constructor never runs because a module was discovered.

### The registry

`ModuleRegistry` is immutable, built from a `ModuleSource` when the panel boots.

| Lookup | Returns |
|---|---|
| `manifest(string $ident)` | One module's manifest. |
| `registrar(string $ident)` | One module's registrar metadata. |
| `manifests()` | Every manifest. |
| `idents()` | Every ident. |
| `isEnabled()`, `isDisabled()`, `has()` | Whether a module is enabled, disabled, or known at all. |

Which modules are enabled comes from `ModulesEnabled`, read through the configuration registry between its two
seals, per [RFC-0004](0004-toml-configuration-loading.md). The module system never scans a directory to work out
what is enabled, and never repeats the rule that a module configuration file's presence is what enables it:
configuration is the single source of that.

A module present in the source but not in the enabled list is known and disabled. A module in the enabled list with
no manifest is logged and skipped, because an installation whose configuration names a module that is no longer
installed should still boot.

The registry's own lifetime follows the mode that produced it, per [RFC-0007](0007-binding-lifetimes.md):

| Mode | Lifetime | Effect |
|---|---|---|
| Production | `Process` | Built once at worker boot from compiled metadata, and shared by every request. |
| Debug | `Cycle` | Rebuilt for each cycle, so edited module code takes effect immediately. |

The class is the same in both. Only what constructs it differs.

### Lifecycle

Two phases, in order, across every enabled module:

1. **Register.** Each module's registrar is instantiated and its `#[Register]` method is called with an
   `EngineBuilder`, which is where it registers bindings and resolvers.
2. **Boot.** Once every module has registered and the container exists, each module's `#[Boot]` method is called
   with no arguments.

Every module registers before any module boots, so a module's boot may depend on another module having registered.
A module with no method for a phase is skipped for it, without error.

Registrars are instantiated directly during register, because the container does not exist yet: building it is what
the phase produces. By boot the container exists, and the registrars instantiated during register are the same
objects.

#### The builder

`EngineBuilder` is the mutable half of the container's registries, per
[ADR-0003](../adr/0003-mutable-registries-are-sealed-into-immutable-catalogues.md), presented as one surface to a
module:

| Method | Effect |
|---|---|
| `bind(string $abstract): BindingBuilder` | Registers a binding, returning the builder from [RFC-0001](0001-dependency-injection-container.md). |
| `resolver(string $resolvable, string $resolver, bool $default = false)` | Registers a resolver against a resolvable attribute. |

A builder is scoped to a module for the duration of that module's register call, so every binding registered through
it records the ident that registered it. Recording the owner is this design's part; what an owner means at
resolution is module scopes.

After the register phase the builder is consumed, producing the `BindingCatalogue` and `ResolverCatalogue` the
container is constructed with, and is unavailable afterwards. This is the assembly
[RFC-0001](0001-dependency-injection-container.md) leaves to the module lifecycle.

#### Alias chains

An alias may name another alias. Assembly flattens every chain, so each alias names the abstract that owns the
binding directly and resolution stays a single hop.

It is done here because a catalogue is assembled once and never changes afterwards, while an alias is normalised
before the instance cache is consulted on every resolution, per
[RFC-0006](0006-container-improvements.md). Flattening at assembly pays for the walk once, at boot, rather than on
every resolution for a structure that cannot change.

A chain returning to an alias already seen cannot be flattened, and fails the assembly, naming the aliases in the
cycle. A cycle is therefore a boot failure that says what is wrong, rather than a resolution that exhausts the
stack.

### Collection

Everything a module contributes beyond bindings is pulled rather than pushed. A component asks for contributions
when it wants them, which may be at boot, on first use, or never.

A component defines its own collector type and a handler that drives collection for it:

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

The registry drives the flow, and the component decides when it runs:

1. The component passes its handler to `ModuleRegistry::collect(CollectorHandler $handler, ?PanelContext $context)`.
2. The registry walks every enabled module.
3. For each, it checks the registrar metadata for a `#[Collect]` method accepting that collector type, and skips the
   module when there is none.
4. `create()` produces a fresh collector for that module.
5. The module's method populates it.
6. `process()` receives the populated collector.
7. After every module, `finalise()` runs once.

A module receives a fresh collector each time, so nothing a module contributes can reach or overwrite what another
contributed. The handler is the only thing that sees them all.

```php
#[Collect]
public function routes(RouteCollector $routes): void
{
    $routes->get('/backups', ListBackups::class);
}
```

The collector type is taken from the method's type hint. A `#[Collect]` method may also declare a `PanelContext`
parameter, in which case it participates only when collection runs for that context.

#### Scope of a contribution

By default a module's `#[Collect]` method contributes within its own module's boundary, and the `$scoped` flag
passed to the handler says so. `#[Unscoped]` marks a method as contributing outside it.

Whether that is permitted is the handler's decision, never the module system's. The handler receives both the flag
and the module's manifest, so it has the declaration and the capabilities to decide with. Permissions refuse
unscoped contributions outright; routing may allow one from a module declaring `CrossRoutes`.

`#[Unscoped]` here means outside the module's own contribution boundary. It is unrelated to the container's module
scopes, which are a separate axis.

#### Caching what was collected

There is no cacheable collector abstraction. Different components cache differently: a route collector can cache a
form that skips collection entirely on later requests, while another gains nothing from caching at all. A component
caches around collection however suits it.

### Panel context

`PanelContext` names the part of the panel something belongs to: `Account`, `Server` or `Platform`. It is used to
scope collection, and routing uses it to decide a route's URL prefix.

`Server` is a subcontext of `Account` for routing and a value in its own right for collection.

It is defined in the engine's shared values rather than inside collection, alongside the client address value
object. Collection is where it was first written down, but routing needs it too, which is the promotion
[RFC-0010](0010-http-transport.md) records as unsettled. Two consumers make it shared, so it is not owned by
either.

### Errors

Every exception implements `ModuleException`.

| Exception | Thrown when |
|---|---|
| `ModuleSourceException` | Metadata is missing or unreadable. Its message says what to do: run discovery, or check the installation. |
| `UnknownModuleException` | A manifest or registrar is asked for by an ident that is not known. |

An enabled module with no manifest is a warning rather than an exception, and the module is skipped.

### Out of scope

- **Module scopes.** What the owner recorded on a binding means at resolution, and what a module is handed because
  of it, is its own design.
- **Module resources.** Provisioning a module's schema, directories or filesystems.
- **The build step.** Reflecting bundled registrars during the build and writing them into the binary belongs to
  bootstrapping, along with everything else that composes the engine.
- **Enabling and disabling modules**, which is the command line's, per
  [RFC-0004](0004-toml-configuration-loading.md).
- **Which collectors exist.** Each component defines its own collector and handler. This design defines the
  mechanism and nothing that uses it.
- **Enforcing capabilities**, which belongs to whatever a capability governs.
- **A module API package and an SDK** for third-party development.
- **Inter-module dependencies.** Nothing here resolves a dependency between two modules, or orders them by one.

## Alternatives considered

The decisions this design rests on are recorded separately, with the alternatives each one rejected:

- installing modules with Composer driven as a library, in
  [ADR-0012](../adr/0012-modules-are-installed-with-composer-used-as-a-library.md)
- core features being modules, in [ADR-0013](../adr/0013-core-features-are-modules.md)
- sealing mutable registries into immutable catalogues, in
  [ADR-0003](../adr/0003-mutable-registries-are-sealed-into-immutable-catalogues.md)

**Collection as a third lifecycle phase**, with modules pushing every contribution during boot. A component would
then receive contributions whether or not it is ever used, and would have nowhere to put the decision of when to
collect. Pulling lets a component collect on first use, or never.

**A cacheable collector abstraction.** Components cache collected data in different shapes, and some gain nothing
from caching, so an abstraction would have to cover cases that do not resemble each other.

**Following alias chains at runtime**, walking from one alias to the next on each lookup. The walk would repeat on
every resolution, in the path taken before the instance cache is consulted, for a structure fixed at boot. A cycle
would also be found by exhausting the stack rather than when the catalogue was assembled.

No other alternatives were weighed.

## Backwards compatibility

Nothing breaks. No module exists, and the modules directory in the repository is empty.

The container is constructed with its catalogues today and gains the step that builds them, which is what
[RFC-0001](0001-dependency-injection-container.md) left to this design rather than a change to it.

## Open questions

## Changelog

- 2026-09-16: Alias chains are flattened when the catalogue is assembled, which settles the only open question.

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
