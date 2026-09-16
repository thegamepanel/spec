---
id: ADR-0008
title: Database access is built directly on PDO
status: accepted
created: 2026-03-29
decided: 2026-04-17
backfilled: 2026-09-14
depends: []
updates: []
obsoletes: []
---

# ADR-0008: Database access is built directly on PDO

## Context

The panel needs database access: connections, a way to build queries, and a way to build and change the schema.
Existing database abstraction layers provide all of that, but each one pulls in a great deal of code and a large
number of third-party libraries, which adds to the panel's footprint and complicates it.

## Decision

The database component is built directly on PHP's PDO extension, with its own connections, query builder and schema
builder, rather than on an existing database abstraction layer.

## Alternatives

**An existing database abstraction layer.** Every one available pulls in so much code and so many third-party
libraries that it complicates the panel and adds to its footprint.

No other alternatives were weighed.

## Consequences

Easier:

- The panel's footprint stays small. The database component needs the PDO extension and no further libraries.
- The component's API follows the panel's own conventions, such as configuration held in typed objects and
  connections injected through an attribute.

Harder:

- The component owns everything a database abstraction layer would otherwise provide, and its correctness:
  connection handling, differences between drivers, SQL generation, identifier quoting and bindings.

Constrained:

- Supporting another database means writing that support inside the component, as the hidden driver support in
  [ADR-0007](0007-mysql-and-mariadb-are-the-default-database.md) anticipates.

## Sources

- Commit [4e4b8bf], "Create connection factory", 2026-03-29, on the branch of PR [#30]: the first code creating
  connections directly with PDO, which is `created`.
- PR [#30], feat(database): Add the database component, merged 2026-04-17 and squashed as [560e9ab]: the component
  built on PDO, which is `decided`.
- [`composer.json` at 560e9ab](https://github.com/thegamepanel/panel/blob/560e9ab/composer.json): the PDO extension
  required, and no database abstraction layer.
- The decision and its reason, that existing database abstraction layers pull in so much code and so many third-party
  libraries that they complicate the panel and add to its footprint: first written down on 2026-09-14, with no record
  of when they were weighed.
- Issues [#41], [#42] and [#48], 2026-08-07 and 2026-08-08: defects in identifier quoting, bindings and connection
  options, which show the correctness the component owns, taken from what followed rather than anticipated.

[4e4b8bf]: https://github.com/thegamepanel/panel/commit/4e4b8bf
[#30]: https://github.com/thegamepanel/panel/pull/30
[560e9ab]: https://github.com/thegamepanel/panel/commit/560e9ab
[#41]: https://github.com/thegamepanel/panel/issues/41
[#42]: https://github.com/thegamepanel/panel/issues/42
[#48]: https://github.com/thegamepanel/panel/issues/48
