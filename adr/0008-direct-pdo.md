---
id: ADR-0008
title: Build database access directly on PDO
status: accepted
created: 2026-03-29
decided: 2026-04-17
backfilled: 2026-11-18
depends: []
updates: []
obsoletes: []
---

# ADR-0008: Build database access directly on PDO

## Context

The panel needs database access: [connection](../GLOSSARY.md#connection)s, a way to build queries, and a way to build
and change the schema. Existing database abstraction layers provide all of that, but each one brings a great deal of
code and a large number of third-party libraries with it, which adds to the panel's footprint and complicates it.

## Decision

I will build the database component directly on PHP's PDO extension, with its own
[connection](../GLOSSARY.md#connection)s, query builder and schema builder, rather than on an existing database
abstraction layer.

## Alternatives

**An existing database abstraction layer**, providing connections, query building and schema building ready-made. I
rejected this because every one available pulls in so much code and so many third-party libraries that it
complicates the panel and adds to its footprint.

## Consequences

Easier:

- The panel's footprint will stay small. The database component will need the PDO extension and no further
  libraries.
- The component's API will follow the panel's own conventions, such as configuration held in typed objects and
  [connection](../GLOSSARY.md#connection)s injected through an attribute.

Harder:

- The component will own everything a database abstraction layer would otherwise provide, and its correctness:
  connection handling, differences between drivers, SQL generation, identifier quoting and
  bound values.

Constrained:

- Supporting another database will mean writing that support inside the component, as the hidden driver support in
  [ADR-0007](0007-mysql-and-mariadb.md) anticipates.

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
