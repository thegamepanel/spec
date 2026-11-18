---
id: ADR-0012
title: Install modules with Composer used as a library
status: accepted
created: 2026-06-01
decided: 2026-06-01
backfilled: 2026-11-18
depends: [ADR-0009]
updates: []
obsoletes: []
---

# ADR-0012: Install modules with Composer used as a library

## Context

[Module](../GLOSSARY.md#module)s come from two places. A bundled module ships inside
the binary as part of the panel's own codebase, as
[ADR-0009](0009-frankenphp-worker.md) decides for the whole application. An
external module is installed alongside the binary. External modules are Composer
packages with dependencies of their own, and some of those dependencies are packages the panel already provides.

The panel depends on `composer/composer`, added for installing external modules.

## Decision

I will make an external module a Composer package of type `tgp-module`, installed into
a Composer project of its own in the modules directory, with its own `composer.json` and `vendor/`. The panel will drive
Composer as a PHP library rather than running it as a standalone tool, and will include the modules project's autoloader
alongside its own.

## Alternatives

**Running Composer as a standalone tool.** I rejected this because using Composer as a library gives the panel full
programmatic control over resolving repositories and installing packages.

## Consequences

Easier:

- The panel will have full programmatic control over resolving repositories and installing
  [module](../GLOSSARY.md#module)s.
- An external module will declare its dependencies as ordinary Composer packages, and
  Composer will not install the ones the panel already provides.

Harder:

- Composer will be a runtime library of the panel, so its vulnerabilities will be the panel's to patch.
- Composer's own extension requirements will become requirements of the binary.
- The modules project's `provide` block will have to be generated from the panel's own lock file, and kept in step
  whenever modules are added, removed or updated.

Constrained:

- Module metadata that Composer's schema does not carry will live under `extra.tgp` in each
  module's `composer.json`.

## Sources

- Issue [#35], Engine - Modules, original text of 2026-06-01 in its edit history: the decision, which is both `created`
  and `decided`, the reason for using Composer as a library, the separate Composer project, the modules autoloader and
  the `provide` block.
- Issue [#36], Engine - Modules - Discovery and Manifest, 2026-06-02: packages of type `tgp-module`, and metadata under
  `extra.tgp`.
- [`composer.json` at 7fad8cd](https://github.com/thegamepanel/panel/blob/7fad8cd/composer.json), the initial commit:
  `composer/composer`, added for installing modules before the design was written.
- PR [#75], chore: Rename package, add licence and patch Composer CVE, merged 2026-09-13 and squashed as [3372f20]:
  Composer's vulnerabilities and extension requirements becoming the panel's, taken from what followed rather than
  anticipated. `composer/composer` was updated to 2.10.3 because 2.10.2 is affected by CVE-2026-84361, and 2.10.3
  declares `ext-filter` and `ext-hash`, which the single-binary build will need to compile in.

[#35]: https://github.com/thegamepanel/panel/issues/35
[#36]: https://github.com/thegamepanel/panel/issues/36
[#75]: https://github.com/thegamepanel/panel/pull/75
[3372f20]: https://github.com/thegamepanel/panel/commit/3372f20
