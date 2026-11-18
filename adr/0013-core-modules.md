---
id: ADR-0013
title: Build core features as modules
status: accepted
created: 2026-06-01
decided: 2026-06-01
backfilled: 2026-11-18
depends: []
updates: []
obsoletes: []
---

# ADR-0013: Build core features as modules

## Context

The [engine](../GLOSSARY.md#engine) is the foundation the rest of the panel is built on, and the module system is its
extension mechanism, open to first-party and third-party [module](../GLOSSARY.md#module)s alike.

Some of the panel's functionality is low-level framework machinery that everything else depends on. Other features are
more complex: they typically combine several components, and many of them are heavy on configuration or face the user.

## Decision

I will keep the [engine](../GLOSSARY.md#engine) to base framework functionality, the low-level parts everything else
depends on, and build every feature that combines several components, especially one heavy on configuration or facing
the user, as a [module](../GLOSSARY.md#module) on the same module system third parties use. Auth and servers will be
modules.

## Alternatives

No alternatives were weighed.

## Consequences

Easier:

- The [engine](../GLOSSARY.md#engine) will stay limited to the low-level parts that everything else depends on.

Harder:

- Engine components will provide mechanism without meaning, and leave the meaning to [module](../GLOSSARY.md#module)s.

Constrained:

- First-party features will ship as bundled modules: inside the binary, with their manifests declared in code and a
  core flag set.

## Sources

- Brainstorming session, date not recalled, not publicly available and first written down on 2026-09-13: the reasoning
  for the decision.
- Issue [#35], Engine - Modules, original text of 2026-06-01 in its edit history: the first written record of the
  decision. It is both `created` and `decided`, because the date of the brainstorming session is not recalled. It also
  names permissions as a module; whether permissions will be a module of its own is not settled.
- Issue [#36], Engine - Modules - Discovery and Manifest, 2026-06-02: bundled modules with manifests declared in code
  and a core flag.

[#35]: https://github.com/thegamepanel/panel/issues/35
[#36]: https://github.com/thegamepanel/panel/issues/36
