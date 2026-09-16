---
title: Glossary
includes: [ADR-0002, ADR-0003, RFC-0001, RFC-0002, RFC-0003, RFC-0004, RFC-0005]
---

# Glossary

<!-- One term per entry, alphabetical. Present tense. Terms are what things are called now; the RFC that named or renamed a term belongs in includes. -->

## Binding

Registered resolution information for one class or interface in the dependency injection container: the concrete class to construct in its place, an instance, or a factory, together with its aliases, its named and qualified child bindings, and whether it is shared, liminal or lazy.

## Catalogue

An immutable, read-only collection consumed at runtime, the counterpart of a registry. The configuration catalogue is sealed from the configuration registry, and the dependency injection container is constructed with a binding catalogue and a resolver catalogue.

## Configuration object

An instance of a class unique to one configuration, holding that configuration's values as typed properties. It is registered against a module and a name, its class maps to exactly one module and name, and the configuration catalogue looks it up by either.

## Connection

An open connection to one database, identified by the name it is configured and requested under, wrapping a PDO connection. It executes queries and statements that write, and manages transactions.

## Cursor

The rows returned by a query, fetched one at a time as they are iterated, so that a large result is processed without every row being held in memory at once. Its rows can be iterated once.

## Expression

An object that produces SQL, with a `?` placeholder for each bound value, together with the values bound to those placeholders. Every object in the query builder and the schema builder is one, and an expression containing other expressions builds its SQL from theirs and gathers their bound values in the same order.

## Ghost object

An object of the final class that the dependency injection container creates uninitialised. When it is first accessed, its constructor runs on that same object through the container, and from then on it is the instance itself rather than a stand-in for one. A dependency is resolved as a ghost object with the `Ghost` attribute.

## Invocation

Calling a callable, method or constructor with its parameters supplied by the dependency injection container, and the object that describes such a call, including any arguments supplied up front by parameter name.

## Lazy proxy

A stand-in that the dependency injection container returns in place of an instance not yet resolved. When the proxy is first used, the container resolves the real instance, and the proxy forwards to it from then on, remaining a proxy.

## Liminal

Describes an instance that is shared but held by the dependency injection container through a weak reference, so that garbage collection can clear it once nothing else uses it, after which the next resolution produces a new instance.

## Primary connection

The connection used whenever no connection name is given. Its name is configured, and a connection must be configured under that name.

## Qualifier

An attribute implementing the `Qualifier` contract that selects a qualified binding for a dependency. The qualifier's class is the key: it selects the binding, any value the attribute carries is ignored, and a qualifier class is unique to one binding outcome.

## Registry

The mutable counterpart of a catalogue, used while things are being registered. Configuration registrations are sealed into the configuration catalogue, and bindings and resolvers each have a registry alongside their catalogue.

## Resolution

Producing an instance of a class through the dependency injection container, from a binding or automatically from its constructor, and the object that describes such a request: the class, any constructor arguments, a name or qualifier, and whether the result is lazy or liminal.

## Resolvable attribute

A parameter attribute that selects a custom resolver for the dependency it is placed on, paired with that resolver in the resolver catalogue. `Ghost`, resolved by `GhostResolver`, is one.

## Resolver

An object implementing the `Resolver` contract that resolves a dependency for the dependency injection container. A generic resolver handles dependencies by their type by default, and a resolvable attribute selects a custom resolver instead.

## Result

The rows returned by a query, fetched in full the first time any row is read and then kept.

## Root

An absolute directory under which the engine keeps one kind of file. `Paths` holds the roots and joins relative paths onto them, and does not work out where they are, check that they exist, or create them.

## Row

One row of a result, holding its values keyed by column name, with typed accessors that cast a value and throw when it cannot be cast.
