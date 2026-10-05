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

We will render HTML on the server from Twig templates, drawn from an ordered [source chain](../GLOSSARY.md#source-chain)
in which later registration wins, so a theme or a [module](../GLOSSARY.md#module) can replace any template. Resolving a
[template reference](../GLOSSARY.md#template-reference) will report which
[template source](../GLOSSARY.md#template-source) answered it. A [slot](../GLOSSARY.md#slot) will let other code
contribute to a page without the page knowing who contributed.

## Motivation

The panel is server-rendered, as [ADR-0001](../adr/0001-the-panel-is-a-single-system-not-a-panel-and-an-engine.md)
records, and nothing in the [engine](../GLOSSARY.md#engine) turns a template into HTML.

Templates do not come from one place. Core provides them, each [module](../GLOSSARY.md#module) provides its own, and a
theme replaces any of them without owning a copy of everything it does not change. Pages also have to be extensible: a
module adds a tab to a page core owns, without core knowing the module exists.

## Proposal

### Sources

A [template source](../GLOSSARY.md#template-source) will be a `Source`: a name and a map from each
[template namespace](../GLOSSARY.md#template-namespace) to a path. The name will exist for provenance; the pairs will be
what resolution uses.

```
core      main     -> <core templates>
backups   backups  -> <the backups module's views>
midnight  main     -> <the midnight theme's main templates>
          backups  -> <the midnight theme's backups templates>
```

A [module](../GLOSSARY.md#module) will register one pair, for its own namespace, and its files will sit directly beneath
that path, so `views/list.twig` will be `backups:list`. A theme will register one pair for each namespace it overrides.
A theme that provides two files across two namespaces will provide nothing else, and everything it does not provide will
fall through to whichever source does.

Registration will be explicit. Nothing will scan a directory or infer a namespace from a layout, because a module and
a theme are laid out differently and any convention covering both would encode what a theme looks like into a
component that must not know what a theme is.

Sources will be collected in a [registry](../GLOSSARY.md#registry), `SourceRegistry`, and sealed into a
[catalogue](../GLOSSARY.md#catalogue), `SourceCatalogue`, following
[ADR-0003](../adr/0003-mutable-registries-are-sealed-into-immutable-catalogues.md).

Core will register itself as a source like anything else. There will be no base path with special cases layered over it,
which is what settles where core's templates live: they will live with core, and core will register a source for them.

### Precedence

Later registration will win. Core will register first, because it boots first and cannot know what follows; modules will
register next; a theme will register last. Registering a [template source](../GLOSSARY.md#template-source) will put its
paths ahead of those already registered, so the most recently registered source will be tried first.

Nothing here will know what a theme or a [module](../GLOSSARY.md#module) is. A theme will be a source registered late,
and a module will be a source registered under its own namespace. The test of that separation will be that a second
source can be registered and have `main:server.view` resolve from it in preference to core's copy, with nothing in this
design naming either concept.

### References

A [template reference](../GLOSSARY.md#template-reference) will always take the form `namespace:path`. The
[template namespace](../GLOSSARY.md#template-namespace) says who owns the template, not which file provides it, so a
[template source](../GLOSSARY.md#template-source) registered later may answer it. That is what will let a theme restyle
a [module](../GLOSSARY.md#module)'s pages, which matters because most pages come from modules.

Every reference will walk the [source chain](../GLOSSARY.md#source-chain) and take the first match.

There will be no protected or unoverridable template. A module will not be able to pin its own partials: a theme blocked
from overriding `backups:_row` would override `backups:list` instead and inline the row, which would protect nothing and
leave the theme owning a copy of a whole page it will not keep in step with. Which templates a module treats as stable
will be documentation, like any other public surface.

### Provenance

Resolving a [template reference](../GLOSSARY.md#template-reference) will return a `Resolution`: the template, and the
[template source](../GLOSSARY.md#template-source) that provided it, not only the file. Twig's loader does not expose the
source, so the component will map a resolved file back to the source root containing it. Provenance will answer the
first question anyone debugging a [source chain](../GLOSSARY.md#source-chain) asks, and adding it later would change the
shape of what resolution returns.

### Rendering

```php
$renderer->render('main:server.view', $data);
$renderer->renderBlock('main:server.view', 'console', $data);
```

`Renderer` will render a [template reference](../GLOSSARY.md#template-reference), or one block of it, to HTML.
`render()` will produce the whole template. `renderBlock()` will produce one named block of it, with no layout around
it, which is what a hypermedia interaction swapping part of a page needs.

Whether a request wants a page or a block will be decided by whatever maps a result onto a response, since that is
what knows about headers, as [RFC-0010](0010-http-transport.md) describes.

A template will receive a [read model](../GLOSSARY.md#read-model) carrying the fields its page needs, not an entity, so
a template handed a server will not be able to reach `server.owner.email`: `owner` will not be there to reach.

Twig's sandbox will be configured as well. A read model bounds what a template is given; the sandbox bounds what it
may do with what it is given, and neither substitutes for the other. What the sandbox permits will be decided for each
[module](../GLOSSARY.md#module) rather than centrally, since a module knows what its own templates need.

### Slots

A [slot](../GLOSSARY.md#slot) will be a named point in a template where other code contributes. It will be a Twig tag
rather than a function, so it will carry default content and an explicit payload:

```twig
{% slot 'server.tabs' with { server: server } %}
    No additional tabs.
{% endslot %}
```

`SlotRegistry` will collect slot definitions and seal them into `SlotCatalogue`, the sealed snapshot rendering reads. A
definition will carry the slot's name, a [template reference](../GLOSSARY.md#template-reference), and an optional
condition. Conditions will be evaluated when the slot renders, so a contribution can depend on a permission or on state,
and nothing will be dispatched for each render beyond that.

Definitions will append, so contributions will never conflict, and they will render in registration order.

A contribution will receive the declared payload and nothing else. It will not inherit the page's context, so a slot
will be a name plus a payload shape: a contract a [module](../GLOSSARY.md#module) can write against, rather than
whatever happened to be in scope at that point in that template.

Slot names will be namespaced by whoever declares them. Core will declare the core vocabulary, and anything else with
code will declare its own under its own namespace, so a module depending on another module's slot will be an ordinary
declared dependency. A template will be able to reference a slot but not declare one, because declaring will be a code
path, so a [template source](../GLOSSARY.md#template-source) contributing only templates will not be able to extend the
vocabulary.

Blocks and slots will be independent of each other. A block will be a region marked by whoever wrote the template; a
slot will be a point others contribute to. Either may contain the other, and slots will take no part in rendering a
block.

### The environment

The Twig environment will be built once when the panel boots, from the sealed source
[catalogue](../GLOSSARY.md#catalogue), and reused for the life of the worker, as
[ADR-0009](../adr/0009-the-panel-runs-as-a-frankenphp-worker-in-a-single-binary.md) records. The registries will be
mutable while the panel boots and sealed afterwards. In debug, templates will reload when they change, so an edited
template will recompile without a restart.

Compiled templates are PHP that is included and opcached, so they will be written to the `compiled`
[root](../GLOSSARY.md#root), as [RFC-0008](0008-filesystems.md) describes, rather than through a filesystem or alongside
the templates themselves. Core's templates ship inside the binary, where the tree is read only, so compiling beside them
would work while developing and fail once installed.

The compiled cache key will include the identity of the [source chain](../GLOSSARY.md#source-chain) that resolved the
template. With one chain that changes nothing; if more than one ever exists, it will be what stops output compiled under
one chain being served under another.

### Failing closed

- A [template source](../GLOSSARY.md#template-source) registering a path that does not exist will fail when the
  catalogue is sealed, rather than on the first render that needs it.
- A [template reference](../GLOSSARY.md#template-reference) that resolves to nothing will throw, rather than render an
  empty string.
- A [slot](../GLOSSARY.md#slot) with no contributions will not be a failure. Most slots on most pages will be empty, so
  it will render its default body if it has one, and nothing otherwise.

### Out of scope

- **Themes.** Theme manifests, discovery, switching and validation sit above this. A theme is only a
  [template source](../GLOSSARY.md#template-source) registered late.
- **Design tokens, CSS and asset building.**
- **Collecting sources or slot definitions from modules**, which is a [collector](../GLOSSARY.md#collector) in the
  [module](../GLOSSARY.md#module) system.
- **What each module's sandbox permits.** That is decided with the module rather than here.
- **Branding configuration.**
- **Hypermedia response headers.** These are headers, set by whatever builds the response.

## Alternatives considered

The decisions this design rests on are recorded separately, with the alternatives each one rejected:

- sealing mutable registries into immutable catalogues, in
  [ADR-0003](../adr/0003-mutable-registries-are-sealed-into-immutable-catalogues.md)
- Twig as the template engine, in [ADR-0019](../adr/0019-twig-is-the-template-engine.md)

**Protecting a module's own templates from being overridden.** We rejected this because a theme blocked from
overriding a partial overrides the page that contains it instead, which protects nothing and costs the theme a copy of
a whole page.

**Inferring namespaces by scanning directories.** We rejected this because a [module](../GLOSSARY.md#module) and a theme
are laid out differently, so any convention covering both would put knowledge of what a theme is into a component that
must not have it.

**Giving contributions the page's context.** We rejected this because a contribution reaching a variable that core
passes on one page only is a coupling nothing declares and nothing checks.

No other alternatives were weighed.

## Backwards compatibility

Nothing will break. The [engine](../GLOSSARY.md#engine) renders no HTML today.

## Open questions

## Changelog

- 2026-09-16: A sandbox is configured, with what it permits decided per module. This document previously recorded a
  sandbox as unnecessary, following [#69].
- 2026-10-05: Reworded in the readable house style, and the project's own vocabulary linked to the glossary. The
  design is unchanged.

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
- A sandbox being configured, and what it permits being decided per module: first written down on 2026-09-16. Issue
  [#69] states the opposite, that read models make a sandbox unnecessary and that none is configured, and is
  superseded on this point. Read models remain, for the reason [#69] gives.

[#69]: https://github.com/thegamepanel/panel/issues/69
[#71]: https://github.com/thegamepanel/panel/issues/71
