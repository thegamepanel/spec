---
id: ADR-0012
title: Modules are installed with Composer used as a library
status: accepted
created: 2026-06-01
decided: 2026-06-01
backfilled: 2026-09-14
depends: [ADR-0009]
updates: []
obsoletes: []
---

# ADR-0012: Modules are installed with Composer used as a library

## Context

Modules come from two sources: bundled modules, shipped inside the binary as part of the panel's own codebase, per
[ADR-0009](0009-the-panel-runs-as-a-frankenphp-worker-in-a-single-binary.md), and external modules, installed
alongside it. External modules are Composer packages with dependencies of their own,
some of which the panel already provides. The panel depends on `composer/composer`, added for installing them.

## Decision

External modules are Composer packages of type `tgp-module`, installed into a separate Composer project in the
modules directory, with its own `composer.json` and `vendor/`. The panel drives Composer as a PHP library rather than
running it as a standalone tool, and includes the modules autoloader alongside its own.

## Alternatives

**Composer as a standalone tool.** Using it as a library instead gives full programmatic control over repository
resolution and installation.

No other way of installing modules was weighed.

## Consequences

Easier:

- The panel will have full programmatic control over resolving repositories and installing modules.
- External modules will declare their dependencies as ordinary Composer packages, and Composer will not install the
  ones the panel already provides.

Harder:

- Composer will be a runtime library of the panel, so its vulnerabilities will be the panel's to patch.
- Composer's own extension requirements will become requirements of the binary.
- The modules project's `provide` block will need generating from the panel's own lock file, and keeping in sync
  whenever modules are added, removed or updated.

Constrained:

- Module metadata that Composer's schema does not carry will live under `extra.tgp` in each module's
  `composer.json`.

## Sources

- Issue [#35], Engine - Modules, original text of 2026-06-01 in its edit history: the decision, which is both
  `created` and `decided`, the reason for using Composer as a library, the separate Composer project, the modules
  autoloader and the `provide` block.
- Issue [#36], Engine - Modules - Discovery and Manifest, 2026-06-02: packages of type `tgp-module`, and metadata
  under `extra.tgp`.
- [`composer.json` at 7fad8cd](https://github.com/thegamepanel/panel/blob/7fad8cd/composer.json), the initial
  commit: `composer/composer`, added for installing modules before the design was written.
- PR [#75], chore: Rename package, add licence and patch Composer CVE, merged 2026-09-13 and squashed as [3372f20]:
  Composer's vulnerabilities and extension requirements becoming the panel's, taken from what followed rather than
  anticipated. `composer/composer` was updated to 2.10.3 because 2.10.2 is affected by CVE-2026-84361, and 2.10.3
  declares `ext-filter` and `ext-hash`, which the single-binary build will need to compile in.

[#35]: https://github.com/thegamepanel/panel/issues/35
[#36]: https://github.com/thegamepanel/panel/issues/36
[#75]: https://github.com/thegamepanel/panel/pull/75
[3372f20]: https://github.com/thegamepanel/panel/commit/3372f20
