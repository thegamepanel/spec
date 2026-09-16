---
id: RFC-0011
title: Views
status: proposed
created: 2026-09-16
decided:
depends: [ADR-0003, ADR-0019, RFC-0007, RFC-0008, RFC-0010]
updates: []
obsoletes: []
---

# RFC-0011: Views

## Abstract

Server-rendered HTML from Twig templates. Templates come from an ordered chain of sources, each a name and a set of
namespace-to-path pairs, with later registration winning, so a theme or a module can replace any template. A
reference names a namespace and a path, resolution reports which source answered it, and named slots let other code
contribute to a page without the page knowing who contributed.

## Motivation

The panel is server-rendered, per
[ADR-0001](../adr/0001-the-panel-is-a-single-system-not-a-panel-and-an-engine.md), and nothing in the engine turns a
template into HTML.

Templates do not come from one place. Core provides them, each module provides its own, and a theme replaces any of
them without owning a copy of everything it does not change. Pages also have to be extensible: a module adds a tab
to a page core owns, without core knowing the module exists.

## Proposal

### Concepts

| Term | Meaning |
|---|---|
| Source | A named set of namespace-to-path pairs, registered as one unit. |
| Namespace | Who owns a template, such as `main` for core or `backups` for a module. |
| Reference | `namespace:path`, naming a template without saying which file provides it. |
| Chain | The registered sources in precedence order, walked until a reference resolves. |
| Slot | A named point in a template that other code contributes to. |
| Block | A named region of a template, renderable on its own. |
| Read model | An object carrying the fields one page needs, passed to a template in place of an entity. |

### Components

| Component | Responsibility |
|---|---|
| `Source` | A name and its namespace-to-path pairs. |
| `SourceRegistry`, `SourceCatalogue` | Collect sources, and the sealed snapshot the environment is built from. |
| `SlotRegistry`, `SlotCatalogue` | Collect slot definitions, and the sealed snapshot rendering reads. |
| `Renderer` | Renders a reference, or one block of it, to HTML. |
| `Resolution` | What resolving a reference returns: the template and the source that provided it. |

### Sources

A source is a name and a map of namespace to path. The name exists for provenance; the pairs are what resolution
uses.

```
core      main     -> <core templates>
backups   backups  -> <the backups module's views>
midnight  main     -> <the midnight theme's main templates>
          backups  -> <the midnight theme's backups templates>
```

A module registers one pair, for its own namespace, and its files sit directly beneath that path, so `views/list.twig`
is `backups:list`. A theme registers one pair for each namespace it overrides. A theme providing two files across two
namespaces provides nothing else, and everything it does not provide falls through to whoever does.

Registration is explicit. Nothing scans a directory or infers a namespace from a layout, because a module and a theme
are laid out differently and any convention covering both would encode what a theme looks like into a component that
must not know what a theme is.

`SourceRegistry` collects sources and seals into `SourceCatalogue`, per
[ADR-0003](../adr/0003-mutable-registries-are-sealed-into-immutable-catalogues.md).

Core registers itself as a source like anything else. There is no base path with special cases layered over it, which
is what settles where core's templates live: they live with core, and core registers a source for them.

### Precedence

Later registration wins. Core registers first, because it boots first and cannot know what follows; modules register
next; a theme registers last. Registering a source puts its paths ahead of those already registered, so the most
recently registered source is tried first.

Nothing here knows what a theme or a module is. A theme is a source registered late, and a module is a source
registered under its own namespace. The property that proves the separation: a second source can be registered and
have `main:server.view` resolve from it in preference to core's copy, with nothing in this design naming either
concept.

### References

One grammar everywhere: `namespace:path`. The namespace says who owns the template, not which file provides it, so a
source registered later may answer it. That is what lets a theme restyle a module's pages, which matters because most
pages come from modules.

Every reference walks the chain and takes the first match.

There is no protected or unoverridable template. A module cannot pin its own partials: a theme blocked from
overriding `backups:_row` overrides `backups:list` instead and inlines the row, which protects nothing and leaves the
theme owning a copy of a whole page it will not keep in step with. Which templates a module treats as stable is
documentation, like any other public surface.

### Provenance

Resolving a reference reports the source that provided the template, not only the file. Twig's loader does not expose
that, so the component maps a resolved file back to the source root containing it. It answers the first question
anyone debugging a chain asks, and adding it later would change the shape of what resolution returns.

### Rendering

```php
$renderer->render('main:server.view', $data);
$renderer->renderBlock('main:server.view', 'console', $data);
```

`render()` produces a whole template. `renderBlock()` produces one named region of it, with no layout around it,
which is what a hypermedia interaction swapping part of a page needs.

Whether a request wants a page or a block is decided by whatever maps a result onto a response, since that is what
knows about headers, per [RFC-0010](0010-http-transport.md).

Templates receive read models carrying the fields a page needs, not entities. That is why no sandbox is configured:
a restriction list would have to forbid a template handed a `Server` from reaching `server.owner.email`, whereas a
read model means `owner` is not there to reach. There is nothing to permit and nothing to keep in step as entities
grow.

### Slots

A slot is a named point in a template where other code contributes. It is a tag rather than a function, so it carries
default content and an explicit payload:

```twig
{% slot 'server.tabs' with { server: server } %}
    No additional tabs.
{% endslot %}
```

`SlotRegistry` collects definitions and seals into `SlotCatalogue`. A definition carries the slot's name, a template
reference, and an optional condition. Conditions are evaluated when the slot renders, so a contribution can depend on
a permission or on state, and nothing is dispatched for each render beyond that.

Definitions append, so contributions never conflict, and they render in registration order.

A contribution receives the declared payload and nothing else. It does not inherit the page's context, so a slot is a
name plus a payload shape, which is a contract a module can write against, rather than whatever happened to be in
scope at that point in that template.

Slot names are namespaced by whoever declares them. Core declares the core vocabulary, and anything else with code
declares its own under its own namespace, so a module depending on another module's slot is an ordinary declared
dependency. A template may reference a slot but cannot declare one, because declaring is a code path, so a source
contributing only templates cannot extend the vocabulary.

Blocks and slots are orthogonal. A block is a region marked by whoever wrote the template; a slot is a point others
contribute to. Either may contain the other, and slots take no part in rendering a block.

### The environment

The Twig environment is built once when the panel boots, from the sealed source catalogue, and reused for the life of
the worker, per [ADR-0009](../adr/0009-the-panel-runs-as-a-frankenphp-worker-in-a-single-binary.md). The registries
are mutable while the panel boots and sealed afterwards. In debug, templates reload when they change, so an edited
template recompiles without a restart.

Compiled templates are PHP that is included and opcached, so they are written to the `compiled` root, per
[RFC-0008](0008-filesystems.md), rather than through a filesystem or alongside the templates themselves. Core's
templates ship inside the binary, where the tree is read only, so compiling beside them would work while developing
and fail once installed.

The compiled cache key includes the identity of the chain that resolved the template. With one chain that changes
nothing; if more than one ever exists, it is what stops output compiled under one chain being served under another.

### Failing closed

- A source registering a path that does not exist fails when the catalogue is sealed, rather than on the first render
  that needs it.
- A reference that resolves to nothing throws, rather than rendering an empty string.
- A slot with no contributions is not a failure. Most slots on most pages are empty, so it renders its default body
  if it has one, and nothing otherwise.

### Out of scope

- **Themes.** Theme manifests, discovery, switching and validation sit above this. A theme is only a source
  registered late.
- **Design tokens, CSS and asset building.**
- **Collecting sources or slot definitions from modules**, which is a collector in the module system.
- **A sandbox**, which read models make unnecessary.
- **Branding configuration.**
- **Hypermedia response headers**, which are headers, set by whatever builds the response.

## Alternatives considered

The decision this design rests on is recorded separately, with the alternatives it rejected:

- Twig as the template engine, in [ADR-0019](../adr/0019-twig-is-the-template-engine.md)

**Protecting a module's own templates from being overridden.** A theme blocked from overriding a partial overrides
the page that contains it instead, which protects nothing and costs the theme a copy of a whole page.

**Inferring namespaces by scanning directories.** A module and a theme are laid out differently, so any convention
covering both would put knowledge of what a theme is into a component that must not have it.

**Giving contributions the page's context.** A contribution reaching a variable that core passes on one page only is
a coupling nothing declares and nothing checks.

No other alternatives were weighed.

## Backwards compatibility

Nothing breaks. The engine renders no HTML before this.

## Open questions

## Changelog

## Sources

- Issue [#69], Engine - Views, 2026-08-25, with no edits and no comments: sources as a name plus namespace-to-path
  pairs, explicit registration, core registering as a source like anything else, later registration winning, the
  `namespace:path` grammar and first-match resolution, no protected templates and why, provenance reporting, the
  environment built once with auto-reload in debug, the compiled cache key including the chain's identity, failing
  closed on seal and on an unresolvable reference, empty slots rendering their default, slots as a tag with a payload
  and conditions evaluated at render, contributions receiving only the declared payload, namespaced slot names,
  blocks rendered on their own, read models rather than entities, and every boundary listed under Out of scope.
- Issue [#71], Engine - Filesystems, 2026-09-03: core templates living with core rather than at a path of their own,
  and compiled output belonging at the `compiled` root because PHP includes it. #69 left where core templates live
  open, and this settles it.

[#69]: https://github.com/thegamepanel/panel/issues/69
[#71]: https://github.com/thegamepanel/panel/issues/71
