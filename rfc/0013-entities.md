---
id: RFC-0013
title: Entities
status: proposed
created: 2026-09-16
decided:
depends: [ADR-0011, RFC-0001, RFC-0007, RFC-0012]
updates: []
obsoletes: []
---

# RFC-0013: Entities

## Abstract

A thin layer over the database component: an identifier type per entity, a map holding the entities loaded during one
unit of work along with the data they were loaded from, and a base store that finds, saves and deletes them. Saving
compares an entity against the data it was loaded from and writes only what changed. It is not an object-relational
mapper: turning a row into an entity, and an entity back into columns, belongs to each store.

## Motivation

Working with the database means writing the same things repeatedly: reading a row into an object, deciding whether
saving that object is an insert or an update, working out which columns actually changed, and keeping two objects
loaded from the same row from drifting apart.

None of that needs a full object-relational mapper. It needs identity, change detection, and somewhere to put the
persistence logic that every store would otherwise repeat.

## Proposal

### Concepts

| Term | Meaning |
|---|---|
| Entity | A plain object with an identity, holding data and no persistence logic. |
| Identifier | An entity's identity, a ULID with a type of its own per entity. |
| Entity map | The entities loaded or saved during one unit of work, with the data each came from. |
| Snapshot | The column data an entity was last loaded from or saved as. |
| Store | What finds, saves and deletes entities of one type. |
| Soft deletable | An entity that is marked deleted rather than removed. |

### Components

| Component | Responsibility |
|---|---|
| `Entity` | Contract requiring an identifier. |
| `EntityId` | Abstract identifier, one subclass per entity type. |
| `EntityMap` | Holds the entities of one unit of work, with their snapshots. |
| `EntityStore` | Contract for finding, saving and deleting entities. |
| `BaseEntityStore` | The persistence logic every store shares. |
| `HasTimestamps`, `Timestamps` | Contract and collection for entities carrying timestamps. |
| `IsSoftDeletable` | Contract for entities marked deleted rather than removed. |

### Identity

`EntityId` is an abstract readonly identifier, and each entity type declares its own subclass, so one entity's
identifier cannot be passed where another's is expected. An identifier holds a ULID, validated when it is
constructed, and `EntityId::make()` generates a new one, per
[ADR-0011](../adr/0011-identifiers-are-ulids-generated-in-php.md).

Because a ULID is generated in PHP, an entity has its identity before it is ever written, so inserting needs nothing
back from the database to know what it just stored.

`Entity` requires only `getId(): EntityId`. An entity holds data and knows nothing about being persisted.

### The entity map

`EntityMap` holds the entities of one unit of work, keyed by entity class and identifier, each with the snapshot it
was loaded or last saved from.

| Method | Effect |
|---|---|
| `add(Entity $entity, array $data): void` | Holds the entity and the data as its snapshot. |
| `has(string $entity, string $id): bool` | Whether an entity of that class and identifier is held. |
| `get(string $entity, string $id): ?Entity` | The entity held, if any. |
| `changes(string $entity, string $id, array $data): array` | The columns in the data that differ from the snapshot, never including the identifier. |
| `forget(string $entity, string $id): void` | Drops the entity and its snapshot. |

Two reads of the same row therefore give the same object, and saving compares against what the row actually held
rather than against what the entity was constructed with.

`changes()` compares each column strictly against the snapshot, so a value of a different type counts as a change,
and compares a structured value, such as one stored as JSON, whole rather than field by field.

### Stores

```php
interface EntityStore
{
    public function find(EntityId $id): ?Entity;

    public function save(Entity $entity): WriteResult;

    public function delete(Entity $entity): WriteResult;
}
```

A concrete store is usually final and extends `BaseEntityStore`, which holds the persistence logic and leaves two
things to the store:

| Method | Responsibility |
|---|---|
| `hydrate(Row $row): Entity` | Builds an entity from a row. |
| `dehydrate(Entity $entity): array` | Turns an entity into columns to write. |

A store is given a connection, per [RFC-0012](0012-postgresql-database-layer.md), and the map. It decides which
tables it reads and writes, since a store may well use more than one.

Stores are not entities' business and entities are not stores' business: nothing on an entity saves it.

#### Finding

`find()` returns the entity held in the map if it is there, and otherwise queries for it, hydrates it, holds it in
the map and returns it. Nothing is found twice.

#### Saving

`save()` decides between inserting and updating by whether the map holds the entity:

- **Held:** the entity is dehydrated, compared against its snapshot, and the columns that changed are written. Columns
  the store protects are dropped from that set, the identifier among them by default. With nothing left to write,
  nothing is executed and an empty result is returned rather than nothing at all, so a caller cannot mistake "no
  change" for "did not run".
- **Not held:** the entity is inserted whole.

After either, the entity is held in the map again with its new snapshot.

An entity constructed by hand with an identifier that already exists is therefore inserted, not updated, because
nothing loaded it. Updating means loading first.

Every write a store makes carries its entity's identifier, so the guard in
[RFC-0012](0012-postgresql-database-layer.md) that refuses an unconditional write never applies to one.

#### Deleting

`delete()` removes the row, or marks the entity deleted when it is soft deletable, and forgets it from the map
afterwards.

### Timestamps

An entity carrying timestamps implements `HasTimestamps`, which exposes a `Timestamps` collection of named,
nullable moments. The base store touches them: creating sets the created and updated moments, and updating sets the
updated one.

| Method | Effect |
|---|---|
| `get(string $name)` | The moment, or nothing when unset. |
| `has(string $name)` | Whether it is set. |
| `add(string $name)` | Sets it to now, unless it is already set. |
| `set(string $name, $moment)` | Sets it to the moment given. |
| `touch(string $name)` | Sets it to now, whatever it held. |

Moments are held as immutable date and time objects carrying a time zone, which matches the database storing them
with one, per [RFC-0012](0012-postgresql-database-layer.md).

`Timestamps` belongs to this design rather than to the database component, because it exists for the entity
lifecycle.

### Soft deleting

An entity opts in by implementing `IsSoftDeletable`:

| Method | Effect |
|---|---|
| `hasBeenDeleted(): bool` | Whether it is marked deleted. |
| `markDeleted($now = null): void` | Marks it deleted, at the moment given or now. |
| `markRestored(): void` | Clears the mark. |

The base store then does the rest, and a concrete store never names the column that records it:

| Method | Effect |
|---|---|
| `find(EntityId $id)` | Excludes entities marked deleted. |
| `findDeleted(EntityId $id)` | Finds one whether or not it is marked deleted. |
| `delete(Entity $entity)` | Marks it deleted and writes that, rather than removing the row. |
| `restore(Entity $entity)` | Clears the mark and writes that. |
| `forceDelete(Entity $entity)` | Removes the row, whether or not the entity is soft deletable. |

The column recording the deletion is the base store's, defaulting to a deleted-at column and overridable by a store
that needs another name. An entity that does not implement the contract is always removed, and the four methods
above that only make sense for one that does throw for it.

### Out of scope

- **Relations.** Nothing loads an entity's related entities, and nothing describes how entities relate.
- **A unit of work.** Saving writes immediately. Nothing queues changes to flush together.
- **Querying beyond finding by identifier.** A store adds whatever queries it needs, hydrating rows through the same
  map so that one row is one entity for the unit of work.
- **Events.** Nothing is dispatched when an entity is saved or deleted.
- **Bulk operations.** Saving or deleting many entities at once is left to the store that needs it.
- **Schema.** Nothing here creates or changes tables.

## Alternatives considered

The decision this design rests on is recorded separately, with the alternative it rejected:

- identifiers as ULIDs generated in PHP, in [ADR-0011](../adr/0011-identifiers-are-ulids-generated-in-php.md)

**A full object-relational mapper.** The complexity of one is not needed, and a data mapper with hydration left to
each store does what the panel wants without describing every relationship in metadata.

**Returning nothing when a save finds no changes.** A caller then cannot tell a save that did nothing from a save
that did not happen, so an empty result is returned instead.

No other alternatives were weighed.

## Backwards compatibility

Nothing breaks. No entity layer exists before this.

## Open questions

- **How the entity map is scoped.** It holds the entities of one unit of work, and the container's cycle lifetime in
  [RFC-0007](0007-binding-lifetimes.md) is the mechanism for that, but nothing yet says the map is bound to it. Under
  a long-lived worker, a map with no scope is a map for the life of the process, which would hand one request's
  entity to the next request.

## Changelog

## Sources

- Issue [#32], Engine - Entities, original text of 2026-04-18 and rewritten on 2026-05-14, in its edit history: the
  layer's shape and that it is not an object-relational mapper, the identifier design, the map with its snapshots
  and its methods, the store contract and the base store with hydration and dehydration left to it, finding through
  the map, saving deciding between insert and update with change detection and protected columns, the empty result
  when nothing changed, deleting with a soft-deletable path, automatic timestamps, and the timestamps collection.
- Issue [#62], Engine - HTTP, 2026-08-21: that nothing in the engine defines what ends a unit of work, so a map with
  no scope is one for the life of the process, which is the open question above.
- A store deciding its own tables and being given a connection; the map being shared for one unit of work; `find()`
  returning nothing when there is no match; an entity built by hand being inserted rather than updated; strict
  comparison of changes; and soft deleting being opted into through the contract with the base store hiding the
  column behind `findDeleted()`, `restore()` and `forceDelete()`, which required a way to clear the mark: first
  written down on 2026-09-16.

[#32]: https://github.com/thegamepanel/panel/issues/32
[#62]: https://github.com/thegamepanel/panel/issues/62
