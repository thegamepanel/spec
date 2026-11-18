---
id: ADR-0010
title: Write configuration in TOML
status: accepted
created: 2026-03-30
decided: 2026-04-22
backfilled: 2026-11-18
depends: [ADR-0009, RFC-0002]
updates: []
obsoletes: []
---

# ADR-0010: Write configuration in TOML

## Context

The panel is distributed as a single binary running FrankenPHP, as
[ADR-0009](0009-frankenphp-worker.md) decides. Its configuration is held in typed
[configuration object](../GLOSSARY.md#configuration-object)s, as [RFC-0002](../rfc/0002-configuration-objects.md)
describes, but nothing decides the format that configuration is written in.

A standalone PHP web application would typically write its configuration as PHP files. A panel distributed as a
binary is not bound to that, and can read its configuration from whichever format suits the job.

Configuration is edited by the sysadmins who install and run the panel, and they should be able to do that without
knowing PHP.

## Decision

I will write configuration in TOML files, which sysadmins edit directly, and the panel will not read configuration
from PHP files.

## Alternatives

**PHP files**, which is what a standalone PHP web application would use. I rejected this because a panel distributed
as a binary does not need them, and editing them requires knowing PHP.

**JSON** and **YAML**, which are widely known, as TOML is. I rejected them because TOML is the simplest of the three,
and the closest to what people are already used to.

## Consequences

Easier:

- Sysadmins will edit configuration without knowing PHP, in a format close to what they already know.

Harder:

- The configuration component will need a TOML parser, and [configuration object](../GLOSSARY.md#configuration-object)s
  will be built from the arrays it produces rather than written as PHP.
- [Module](../GLOSSARY.md#module)s will need a way to define how their configuration objects are loaded from those
  files.
- Secrets will need to be kept out of the files, which are plain text, and supplied from the environment instead.

Constrained:

- Configuration will be limited to the values TOML can express: strings, numbers, booleans, dates and times, arrays
  and tables. Anything else will be built by a configuration object from those.

## Sources

- Issue [#31], Engine - Config - TOML, original text of 2026-03-30 in its edit history: the first written record of
  the decision, which is `created`. It gives the reason that a panel distributed as a binary need not use PHP files,
  and anticipates the TOML parser and the need for modules to define how their configuration is loaded.
- Planning session, 2026-04-22, not publicly available and first written down on 2026-09-13: the decision, which is
  `decided`, the reason that sysadmins should edit configuration without knowing PHP, and supplying secrets from the
  environment through interpolation.
- JSON and YAML as the formats weighed against TOML, and why TOML was chosen: first written down on 2026-09-13, with
  no record of when they were weighed.
- [TOML v1.0.0](https://toml.io/en/v1.0.0): the values TOML can express.

[#31]: https://github.com/thegamepanel/panel/issues/31
