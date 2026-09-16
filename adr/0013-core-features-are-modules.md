---
id: ADR-0013
title: Core features are modules
status: accepted
created: 2026-06-01
decided: 2026-06-01
backfilled: 2026-09-14
depends: []
updates: []
obsoletes: []
---

# ADR-0013: Core features are modules

## Context

The engine is the foundation the rest of the panel builds on, and the module system is its extension mechanism, open
to first-party and third-party modules alike.

Some functionality is low-level framework machinery that everything else depends on. Other features are more
complex: they typically combine several components, and many of them are configuration-heavy or user-facing.

## Decision

The engine holds base framework functionality, the low-level parts that everything else depends on. Features that
combine several components, especially configuration-heavy or user-facing ones, are modules, built on the same
module system available to third parties. Auth and servers are modules.

## Alternatives

No other arrangement was weighed.

## Consequences

Easier:

- The engine will stay limited to the low-level parts that everything else depends on.

Harder:

- Engine components will provide mechanism without meaning, leaving the meaning to modules. Sessions will store what
  auth puts in them and give it no meaning, and long-lived re-authentication will belong to auth rather than to
  sessions.

Constrained:

- First-party features will ship as bundled modules: inside the binary, with manifests declared in code and a core
  flag set.
- `Core` will be ambiguous as a name, since core features are themselves modules.

## Sources

- Brainstorming session, date not recalled, not publicly available and first written down on 2026-09-13: the
  reasoning for the decision.
- Issue [#35], Engine - Modules, original text of 2026-06-01 in its edit history: the first written record of the
  decision. It is both `created` and `decided`, because the date of the brainstorming session is not recalled. It
  also names permissions as a module; whether permissions will be a module of its own is not settled.
- Issue [#36], Engine - Modules - Discovery and Manifest, 2026-06-02: bundled modules with manifests declared in code
  and a core flag.
- Issue [#67], Engine - Sessions, 2026-08-22: sessions giving what auth stores no meaning, and re-authentication
  belonging to auth, taken from the sessions design rather than anticipated.
- Issue [#70], Engine - Container - Module scopes, 2026-09-03: `Core` rejected as an attribute name for being
  ambiguous when core features are modules, taken from that design rather than anticipated.

[#35]: https://github.com/thegamepanel/panel/issues/35
[#36]: https://github.com/thegamepanel/panel/issues/36
[#67]: https://github.com/thegamepanel/panel/issues/67
[#70]: https://github.com/thegamepanel/panel/issues/70
