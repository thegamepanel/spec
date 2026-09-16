---
id: ADR-0010
title: Configuration files are TOML
status: accepted
created: 2026-03-30
decided: 2026-04-22
backfilled: 2026-09-14
depends: [ADR-0009, RFC-0002]
updates: []
obsoletes: []
---

# ADR-0010: Configuration files are TOML

## Context

The panel is distributed as a single binary running FrankenPHP, per
[ADR-0009](0009-the-panel-runs-as-a-frankenphp-worker-in-a-single-binary.md). Its configuration is held in typed
configuration objects, per [RFC-0002](../rfc/0002-configuration-objects.md), but nothing decides the format that
configuration is written in.

A standalone PHP web application would typically write its configuration as PHP files. A panel distributed as a
binary is not bound to that, and can read its configuration from whichever format suits the job.

Configuration is edited by the sysadmins who install and run the panel, and they should be able to do that without
knowing PHP.

## Decision

Configuration is written in TOML files, which sysadmins edit directly. The panel does not read configuration from
PHP files.

## Alternatives

**PHP files.** They are what a standalone PHP web application would use, which a panel distributed as a binary does
not need, and editing them requires knowing PHP.

**JSON** and **YAML**. Like TOML, both are widely known. TOML is chosen over them because it is the simplest of the
three, and the closest to what people are already used to.

## Consequences

Easier:

- Sysadmins will edit configuration without knowing PHP, in a format close to what they already know.

Harder:

- The configuration component will need a TOML parser, and configuration objects will be built from the arrays it
  produces rather than written as PHP.
- Modules will need a way to define how their configuration objects are loaded from those files.
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
