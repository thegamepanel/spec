---
id: ADR-0019
title: Use Twig as the template engine
status: accepted
created: 2026-08-25
decided: 2026-08-25
backfilled: 2026-11-18
depends: [ADR-0001]
updates: []
obsoletes: []
---

# ADR-0019: Use Twig as the template engine

## Context

The panel is one server-rendered system, as [ADR-0001](0001-single-system.md)
decides, so the [engine](../GLOSSARY.md#engine) has to turn templates into HTML.

A page is not always rendered whole. A hypermedia interaction swaps one fragment of a page in place, so a named region
of a template has to be renderable on its own.

Templates come from more than one place. Core provides them, each [module](../GLOSSARY.md#module) provides its own, and
a theme can replace any of them. A template's name therefore has to resolve through an ordered chain of locations rather
than a single directory.

Module authors write templates, so whatever the engine chooses is what they work in.

## Decision

I will use Twig as the panel's template engine. Templates will be Twig templates, found through Twig's own loader, and
rendered through one Twig environment built when the panel boots.

## Alternatives

**Other template engines.** Several were weighed, and the record does not name them. I rejected them in favour of Twig
for its sandboxing and its flexibility, and because [module](../GLOSSARY.md#module) authors are more likely to know Twig
than any alternative.

## Consequences

Easier:

- A named block of a template will render on its own, which is what a hypermedia interaction swapping a fragment
  needs.
- One name will be able to try several locations in order, because Twig's loader already accepts more than one path
  for each namespace, so the chain will need no loader of the panel's own.
- Templates will be restrictable through Twig's sandbox if one is ever accepted from somewhere that is not trusted.

Harder:

- Escaping will be for HTML by default, so a value placed in an attribute, a URL or inline JavaScript will need the
  filter for that context, and getting it wrong will be a security bug rather than a rendering one.

Constrained:

- Twig will be a dependency of the binary.
- Templates will be compiled to PHP, and the compiled output included and opcached, so it will have to be written
  somewhere PHP can include from rather than through an abstraction.

## Sources

- Issue [#69], Engine - Views, 2026-08-25, with no edits and no comments: the decision, which is both `created` and
  `decided`. It records Twig's named blocks giving fragment rendering, its loader accepting several paths for each
  namespace so that the chain of locations needs no custom loader, module authors being more likely to know it than
  any alternative, and its escaping being for HTML by default so that other contexts need an explicit filter. It
  also records that templates are compiled to PHP and where the compiled output goes.
- Several alternatives having been weighed, and Twig being chosen for its sandboxing and its flexibility: first
  written down on 2026-09-16. Which engines were weighed is not recorded.

[#69]: https://github.com/thegamepanel/panel/issues/69
