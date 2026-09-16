---
id: ADR-0019
title: Twig is the template engine
status: accepted
created: 2026-08-25
decided: 2026-08-25
backfilled: 2026-09-16
depends: [ADR-0001]
updates: []
obsoletes: []
---

# ADR-0019: Twig is the template engine

## Context

The panel is one server-rendered system, per
[ADR-0001](0001-the-panel-is-a-single-system-not-a-panel-and-an-engine.md), so the engine has to turn templates into
HTML.

Pages are not rendered whole every time. Hypermedia interactions swap a fragment of a page in place, so a named
region of a template has to be renderable on its own.

Templates come from more than one place. Core provides them, modules provide their own, and a theme replaces any of
them, so a reference has to resolve through an ordered chain of locations rather than one directory.

Module authors write templates, so the engine's choice is one they have to work in.

## Decision

Twig is the panel's template engine. Templates are Twig templates, resolved through Twig's own loader, and rendering
goes through a Twig environment built once when the panel boots.

## Alternatives

**Other template engines.** Several were weighed, though the record does not name them. Twig was chosen for its
sandboxing and its flexibility, and because module authors are more likely to know it than any alternative.

## Consequences

Easier:

- A named region of a template can be rendered on its own, which is what a hypermedia interaction swapping a fragment
  needs.
- Several locations can be tried in order for one reference, because Twig's loader already accepts more than one path
  per namespace, so the chain needs no loader of the panel's own.
- Templates can be restricted through Twig's sandbox if a template is ever accepted from somewhere that is not
  trusted.

Harder:

- Escaping is applied for HTML by default, so a value landing in an attribute, in a URL or in inline JavaScript needs
  the filter for that context, and getting it wrong is a security bug rather than a rendering one.

Constrained:

- Twig becomes a dependency of the binary.
- Templates are compiled to PHP, and compiled output is included and opcached, so it has to be written somewhere PHP
  can include from rather than through an abstraction.

## Sources

- Issue [#69], Engine - Views, 2026-08-25, with no edits and no comments: the decision, which is both `created` and
  `decided`. It records Twig's named blocks giving fragment rendering, its loader accepting several paths per
  namespace so that the resolution chain needs no custom loader, module authors being more likely to know it than
  any alternative, and its escaping being for HTML by default so that other contexts need an explicit filter. It
  also records that templates are compiled to PHP and where the compiled output goes.
- Several alternatives having been weighed, and Twig being chosen for its sandboxing and its flexibility: first
  written down on 2026-09-16. Which engines were weighed is not recorded.

[#69]: https://github.com/thegamepanel/panel/issues/69
