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

We will add a thin layer over the database: an [entity identifier](../GLOSSARY.md#entity-identifier) type for each
entity, an [entity map](../GLOSSARY.md#entity-map) holding the entities loaded during one unit of work along with the
data they were loaded from, and a base [store](../GLOSSARY.md#store) that finds, saves and deletes them. Saving will
compare an entity against the data it was loaded from and write only what changed. The layer will not be an
object-relational mapper: turning a row into an entity, and an entity back into columns, will belong to each store.

## Motivation

Working with the database means writing the same things repeatedly: reading a row into an object, deciding whether
saving that object is an insert or an update, working out which columns actually changed, and keeping two objects
loaded from the same row from drifting apart.

None of that needs a full object-relational mapper. It needs identity, change detection, and somewhere to put the
persistence logic that every [store](../GLOSSARY.md#store) would otherwise repeat.

## Proposal

### Identity

An entity will be a plain object with an identity, holding data and no persistence logic.

Each entity type will declare its own [entity identifier](../GLOSSARY.md#entity-identifier), a subclass of the abstract
readonly `EntityId`, so one entity's identifier cannot be passed where another's is expected. An identifier will hold a
ULID, validated when it is constructed, and `EntityId::make()` will generate a new one, as
[ADR-0011](../adr/0011-identifiers-are-ulids-generated-in-php.md) records.

Because a ULID is generated in PHP, an entity will have its identity before it is ever written, so inserting will need
nothing back from the database to know what it just stored.

`Entity` will require only `getId(): EntityId`. An entity will hold data and know nothing about being persisted.

### The entity map

The [entity map](../GLOSSARY.md#entity-map), `EntityMap`, will hold the entities of one unit of work, keyed by entity
class and identifier, each with the [snapshot](../GLOSSARY.md#snapshot) it was loaded or last saved from.

| Method | Effect |
|---|---|
| `add(Entity $entity, array $data): void` | Holds the entity and the data as its snapshot. |
| `has(string $entity, string $id): bool` | Whether an entity of that class and identifier is held. |
| `get(string $entity, string $id): ?Entity` | The entity held, if any. |
| `changes(string $entity, string $id, array $data): array` | The columns in the data that differ from the snapshot, never including the identifier. |
| `forget(string $entity, string $id): void` | Drops the entity and its snapshot. |

Two reads of the same row will therefore give the same object, and saving will compare against what the row actually
held rather than against what the entity was constructed with.

`changes()` will compare each column strictly against the snapshot, so a value of a different type counts as a
change. It will compare a structured value, such as one stored as JSON, whole rather than field by field.

### Stores

`EntityStore` will be the contract for finding, saving and deleting entities:

```php
interface EntityStore
{
    public function find(EntityId $id): ?Entity;

    public function save(Entity $entity): WriteResult;

    public function delete(Entity $entity): WriteResult;
}
```

A concrete [store](../GLOSSARY.md#store) will usually be final and extend `BaseEntityStore`, which will hold the
persistence logic and leave two things to the store:

| Method | Responsibility |
|---|---|
| `hydrate(Row $row): Entity` | Builds an entity from a row. |
| `dehydrate(Entity $entity): array` | Turns an entity into columns to write. |

A store will be given a [connection](../GLOSSARY.md#connection), as [RFC-0012](0012-postgresql-database-layer.md)
describes, and the [entity map](../GLOSSARY.md#entity-map). It will decide which tables it reads and writes, since a
store may well use more than one.

Stores will not be entities' business, and entities will not be stores' business: nothing on an entity saves it.

#### Finding

`find()` will return the entity held in the [entity map](../GLOSSARY.md#entity-map) if it is there, and otherwise query
for it, hydrate it, hold it in the map and return it. Nothing will be found twice.

#### Saving

`save()` will decide between inserting and updating by whether the [entity map](../GLOSSARY.md#entity-map) holds the
entity:

- **Held:** the entity will be dehydrated and compared against its [snapshot](../GLOSSARY.md#snapshot), and the columns
  that changed will be written. Columns the store protects will be dropped from that set, the identifier among them by
  default. With nothing left to write, nothing will be executed and an empty result will be returned rather than nothing
  at all, so a caller cannot mistake "no change" for "did not run".
- **Not held:** the entity will be inserted whole.

After either, the entity will be held in the map again with its new snapshot.

An entity constructed by hand with an identifier that already exists will therefore be inserted, not updated, because
nothing loaded it. Updating will mean loading first.

Every write a [store](../GLOSSARY.md#store) makes will carry its entity's identifier, so the guard in
[RFC-0012](0012-postgresql-database-layer.md) that refuses an unconditional write will never apply to one.

#### Deleting

`delete()` will remove the row, or mark the entity deleted when it is soft deletable, and forget it from the
[entity map](../GLOSSARY.md#entity-map) afterwards.

### Timestamps

An entity carrying timestamps will implement `HasTimestamps`, which will expose a `Timestamps` collection of named,
nullable moments. The base [store](../GLOSSARY.md#store) will touch them: creating will set the created and updated
moments, and updating will set the updated one.

| Method | Effect |
|---|---|
| `get(string $name)` | The moment, or nothing when unset. |
| `has(string $name)` | Whether it is set. |
| `add(string $name)` | Sets it to now, unless it is already set. |
| `set(string $name, $moment)` | Sets it to the moment given. |
| `touch(string $name)` | Sets it to now, whatever it held. |

Moments will be held as immutable date and time objects carrying a time zone, which matches the database storing them
with one, as [RFC-0012](0012-postgresql-database-layer.md) describes.

`Timestamps` will belong to this design rather than to the database layer, because it exists for the entity
lifecycle.

### Soft deleting

An entity will opt in by implementing `IsSoftDeletable`:

| Method | Effect |
|---|---|
| `hasBeenDeleted(): bool` | Whether it is marked deleted. |
| `markDeleted($now = null): void` | Marks it deleted, at the moment given or now. |
| `markRestored(): void` | Clears the mark. |

The base [store](../GLOSSARY.md#store) will then do the rest, and a concrete store will never name the column that
records it:

| Method | Effect |
|---|---|
| `find(EntityId $id)` | Excludes entities marked deleted. |
| `findDeleted(EntityId $id)` | Finds one whether or not it is marked deleted. |
| `delete(Entity $entity)` | Marks it deleted and writes that, rather than removing the row. |
| `restore(Entity $entity)` | Clears the mark and writes that. |
| `forceDelete(Entity $entity)` | Removes the row, whether or not the entity is soft deletable. |

The column recording the deletion will be the base store's, defaulting to a deleted-at column and overridable by a store
that needs another name. An entity that does not implement the contract will always be removed, and the four methods
above that only make sense for one that does will throw for it.

### Out of scope

- **Relations.** Nothing will load an entity's related entities, and nothing will describe how entities relate.
- **A unit of work.** Saving will write immediately. Nothing will queue changes to flush together.
- **Querying beyond finding by identifier.** A [store](../GLOSSARY.md#store) will add whatever queries it needs,
  hydrating rows through the same [entity map](../GLOSSARY.md#entity-map) so that one row is one entity for the unit of
  work.
- **Events.** Nothing will be dispatched when an entity is saved or deleted.
- **Bulk operations.** Saving or deleting many entities at once is left to the store that needs it.
- **Schema.** Nothing here will create or change tables.

## Alternatives considered

The decision this design rests on is recorded separately, with the alternative it rejected:

- identifiers as ULIDs generated in PHP, in [ADR-0011](../adr/0011-identifiers-are-ulids-generated-in-php.md)

**A full object-relational mapper.** We rejected this because the complexity of one is not needed, and a data mapper
with hydration left to each [store](../GLOSSARY.md#store) does what the panel wants without describing every
relationship in metadata.

**Returning nothing when a save finds no changes.** We rejected this because a caller then cannot tell a save that did
nothing from a save that did not happen, so an empty result will be returned instead.

No other alternatives were weighed.

## Backwards compatibility

Nothing will break. There is no entity layer today.

## Open questions

- **How the entity map is scoped.** It holds the entities of one unit of work, and the container's `Cycle`
  [lifetime](../GLOSSARY.md#lifetime) in [RFC-0007](0007-binding-lifetimes.md) is the mechanism for that, but nothing
  yet says the map is bound to it. Under a long-lived worker, a map with no scope is a map for the life of the process,
  which would hand one request's entity to the next request.

## Changelog

- 2026-10-05: Reworded in the readable house style, and the project's own vocabulary linked to the glossary. The
  design is unchanged.

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
