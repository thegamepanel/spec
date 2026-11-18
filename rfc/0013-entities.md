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

I will add a thin layer over the database for working with entities: an
[entity identifier](../GLOSSARY.md#entity-identifier) type for each entity, an [entity map](../GLOSSARY.md#entity-map)
holding the entities of one unit of work beside the data each was loaded from, and a base [store](../GLOSSARY.md#store)
that finds, saves and deletes them. Saving will compare an entity with the data it was loaded from and write only what
changed. The layer will not be an object-relational mapper: turning a row into an entity, and an entity back into
columns, will be left to each store.

## Motivation

Working with the database means writing the same things over and over: reading a row into an object, deciding whether
saving that object is an insert or an update, working out which columns actually changed, and stopping two objects
loaded from the same row from drifting apart.

None of that needs a full object-relational mapper. It needs identity, change detection, and one place for the
persistence logic every [store](../GLOSSARY.md#store) would otherwise repeat.

## Proposal

### Identity

An entity will be a plain object with an identity. It will hold data and know nothing about being persisted: the only
thing `Entity` will require of it is `getId(): EntityId`.

Each type of entity will declare its own [entity identifier](../GLOSSARY.md#entity-identifier), a subclass of the
abstract readonly `EntityId`, so that a game server's identifier cannot be passed where a user's is expected. An
identifier will hold a ULID, validated when the identifier is constructed, and `EntityId::make()` will generate a new
one, as [ADR-0011](../adr/0011-ulid-identifiers.md) decides.

Because the ULID is generated in PHP, an entity will have its identity before it is ever written, and inserting it will
need nothing back from the database to know what was stored.

### One object per row

The [entity map](../GLOSSARY.md#entity-map), `EntityMap`, will hold the entities of one unit of work, keyed by entity
class and identifier. Beside each it will keep a [snapshot](../GLOSSARY.md#snapshot), the data the entity was loaded
from or last saved as. An entity can be added with its data, looked up or checked for by class and identifier, and
forgotten, which drops the snapshot with it. Two reads of the same row will therefore give the same object, and saving
will compare against what the row actually held rather than against what the entity was constructed with.

The map will also say what changed. Given an entity's class, identifier and current data,
`changes(string $entity, string $id, array $data): array` will return the columns whose values differ from the snapshot,
never including the identifier. Each column will be compared strictly, so a value of a different type counts as a
change, and a structured value, such as one stored as JSON, will be compared whole rather than field by field.

### Stores

A [store](../GLOSSARY.md#store) will find, save and delete entities, through this contract:

```php
interface EntityStore
{
    public function find(EntityId $id): ?Entity;

    public function save(Entity $entity): WriteResult;

    public function delete(Entity $entity): WriteResult;
}
```

A concrete store will usually be final and extend `BaseEntityStore`, which will hold the persistence logic and leave two
things to the store: `hydrate(Row $row): Entity`, which builds an entity from a row, and
`dehydrate(Entity $entity): array`, which turns an entity into the columns to write. A store will be given a
[connection](../GLOSSARY.md#connection), as [RFC-0012](0012-postgresql-database-layer.md) describes, and the
[entity map](../GLOSSARY.md#entity-map). It will decide which tables it reads and writes, since a store may well use
more than one.

Entities and stores will stay out of each other's business: nothing on an entity will save it.

#### Finding

`find()` will return the entity from the [entity map](../GLOSSARY.md#entity-map) if it is held there. Otherwise it will
query for the row, hydrate it, hold the entity in the map and return it, or return nothing when no row matches. Nothing
will be loaded twice.

#### Saving

`save()` will insert or update according to whether the [entity map](../GLOSSARY.md#entity-map) holds the entity.

- **When it is held**, the entity will be dehydrated and compared against its [snapshot](../GLOSSARY.md#snapshot), and
  the columns that changed will be written. Columns the store protects will be dropped from that set first, the
  identifier among them by default. When nothing is left to write, nothing will be executed, and an empty result will be
  returned rather than nothing at all, so that a caller cannot mistake "no change" for "did not run".
- **When it is not held**, the entity will be inserted whole.

Either way, the entity will then be held in the map again with its new snapshot.

An entity constructed by hand with an identifier that already exists will therefore be inserted, not updated, because
nothing loaded it. Updating an entity will mean loading it first.

Every write a [store](../GLOSSARY.md#store) makes will carry its entity's identifier, so the guard in
[RFC-0012](0012-postgresql-database-layer.md) that refuses an unconditional write will never apply to one.

#### Deleting

`delete()` will remove the row, or mark the entity deleted when it is soft deletable, and then forget it from the
[entity map](../GLOSSARY.md#entity-map).

### Timestamps

An entity that carries timestamps will implement `HasTimestamps`, which will expose a `Timestamps` collection of named
moments, each of which may be unset. The base [store](../GLOSSARY.md#store) will maintain them: creating an entity will
set its created and updated moments, and updating it will set the updated one.

A moment can be read, returning nothing when it is unset, and checked for. It can be set to a given moment, or to now in
one of two ways: `add()` will leave a moment that is already set alone, where `touch()` will overwrite whatever it held.

Moments will be immutable date and time objects carrying a time zone, matching the database, which stores them with one,
as [RFC-0012](0012-postgresql-database-layer.md) describes.

`Timestamps` will belong to this layer rather than to the database layer, because it exists for the life of an entity.

### Soft deleting

An entity will opt into soft deleting by implementing `IsSoftDeletable`, and will then carry a deletion mark: the moment
it was marked deleted, or nothing. The contract lets the mark be read, set at a given moment or now, and cleared,
through `hasBeenDeleted()`, `markDeleted()` and `markRestored()`.

The base [store](../GLOSSARY.md#store) will turn that mark into behaviour, so a concrete store will never name the
column that records it. Deleting a soft-deletable entity will set the mark and write it, leaving the row in place, and
an ordinary `find()` will hide marked rows. Anything beyond that will be an explicit path on the store: `findDeleted()`
finds an entity whether or not it is marked, `restore()` clears the mark and writes that, and `forceDelete()` removes
the row outright, whether or not the entity is soft deletable.

The column will be the base store's, defaulting to a deleted-at column, and a store that needs another name can override
it. An entity that does not implement the contract will always be removed, and `findDeleted()` and `restore()` will
throw for it.

### Out of scope

- **Relations.** Nothing will load an entity's related entities, and nothing will describe how entities relate.
- **A unit of work.** Saving will write immediately. Nothing will queue changes to flush together.
- **Querying beyond finding by identifier.** A [store](../GLOSSARY.md#store) will add whatever queries it needs,
  hydrating rows through the same [entity map](../GLOSSARY.md#entity-map) so that one row is one entity for the unit of
  work.

## Alternatives considered

This design rests on [ADR-0011](../adr/0011-ulid-identifiers.md), which generates identifiers as
ULIDs in PHP.

**A full object-relational mapper.** I rejected this because its complexity is not needed: a data mapper that leaves
hydration to each [store](../GLOSSARY.md#store) does what the panel wants without describing every relationship in
metadata.

**Returning nothing when a save finds no changes.** I rejected this because a caller could not then tell a save that did
nothing from a save that did not happen.

## Backwards compatibility

Nothing will break. There is no entity layer today.

## Open questions

### How the entity map is scoped

It holds the entities of one unit of work, and the container's `Cycle` [lifetime](../GLOSSARY.md#lifetime) in
[RFC-0007](0007-binding-lifetimes.md) is the mechanism for that, but nothing yet says the map is bound to it. Under a
long-lived worker, a map with no scope is a map for the life of the process, which would hand one request's entity to
the next request.

## Changelog

- 2026-10-05: Reworded in the readable house style, and the project's own vocabulary linked to the glossary. The
  design is unchanged.
- 2026-11-18: Rewrote the document in full, with no change to the design.
- 2026-11-18: Named `findDeleted()` and `restore()` as the methods that throw for an entity that is not soft
  deletable, where the text had counted four.

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
