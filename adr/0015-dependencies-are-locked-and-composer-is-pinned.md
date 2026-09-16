---
id: ADR-0015
title: Dependencies are locked and Composer is pinned
status: accepted
created: 2026-08-18
decided: 2026-08-19
backfilled: 2026-09-14
depends: []
updates: []
obsoletes: []
---

# ADR-0015: Dependencies are locked and Composer is pinned

## Context

`composer.lock` is gitignored, and every CI job runs `composer self-update && composer install && composer
dump-autoload`. Each run resolves dependencies fresh, against a Composer binary that also moves. A transitive
release can turn the build red overnight with no local change, and PHPStan or php-cs-fixer can behave differently
in CI than they do locally.

Committing the lock file is the standard, recommended practice for a project: everyone who sets up the project, CI
included, then runs on exactly the same dependency versions.

## Decision

`composer.lock` is committed, and CI installs from it with `composer install --no-interaction --no-progress`. CI
runs a pinned Composer version, matching the one used locally, and the pin is changed deliberately.

## Alternatives

**Resolving dependencies fresh on every CI run**, with the lock file ignored and Composer self-updated. It lets
builds break with no local change, and lets tools behave differently in CI than locally.

No other arrangement was weighed.

## Consequences

Easier:

- Every CI job will install the dependency versions in the committed lock file, the same ones used locally, so a
  build will not break without a local change and tools will behave the same in CI as they do locally.
- `composer dump-autoload` will no longer be needed in CI, since `composer install` generates the autoloader.

Harder:

- Updating Composer will be a deliberate change to the pin. A vulnerable Composer will stay pinned until someone
  moves it, so Composer's security releases will need acting on.

Constrained:

- Declared lower bounds will not be tested. A `--prefer-lowest` matrix leg, keeping them honest, will be a separate
  change.

## Sources

- PR [#50], chore(engine): Tooling, CI and documentation cleanup, opened 2026-08-18 and merged 2026-08-19, squashed
  as [54dcfb0]: the problem, the decision, the alternative and the consequences other than Composer's security
  releases. Its commit message and diff record the lock file and pin changes. `created` is when it was opened and
  `decided` when it merged.
- PR [#75], chore: Rename package, add licence and patch Composer CVE, merged 2026-09-13 and squashed as [3372f20]:
  the consequence for Composer's security releases, taken from what followed rather than anticipated. It moved the
  pin from 2.9.3 to 2.10.3 because 2.9.3 is inside the range affected by CVE-2026-84361, which had left CI running
  a Composer open to arbitrary command execution through a package's Perforce source URL.
- Composer documentation, Basic usage, ["Commit your composer.lock file to version control"][composer-lock], read
  2026-09-13: committing the lock file as the standard practice for projects, and why.

[#50]: https://github.com/thegamepanel/panel/pull/50
[#75]: https://github.com/thegamepanel/panel/pull/75
[54dcfb0]: https://github.com/thegamepanel/panel/commit/54dcfb0
[3372f20]: https://github.com/thegamepanel/panel/commit/3372f20
[composer-lock]: https://getcomposer.org/doc/01-basic-usage.md#commit-your-composer-lock-file-to-version-control
