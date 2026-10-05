---
id: RFC-0014
title: Sessions
status: proposed
created: 2026-09-16
decided:
depends: [ADR-0022, RFC-0007, RFC-0010, RFC-0012, RFC-0013]
updates: []
obsoletes: []
---

# RFC-0014: Sessions

## Abstract

We will add server-side sessions, keyed by a cookie and stored in the database, with flash state carrying one value from
the request that writes it to the request that follows. A session will be resolved once per
[cycle](../GLOSSARY.md#cycle), loaded only when something reads it, and written only when something changed. The cookie
will be set on the response during the request, and the record persisted when the cycle closes.

## Motivation

A server-rendered panel needs to remember who is using it between requests, and to carry a message across the
redirect that follows a form submission. Neither is possible with the request alone.

Sessions will be their own component rather than part of the HTTP layer. They will own a table, a store, an expiry
policy and a pruning task, none of which is transport. What is HTTP-facing will be thin: reading a cookie, and writing
one onto a response.

## Proposal

### Storage

A session will live in the database, and the cookie will carry nothing but an identifier. We rejected a
self-contained cookie holding the session's data: it would save a read on every request, but it cannot be revoked.
Suspending someone, or forcing them out, has to invalidate a live session, and a cookie that carries its own data
stays valid until it expires.

`SessionStore` will be the contract for loading, persisting and removing records, and the database implementation will
be the one that exists, named as the entity stores in [RFC-0013](0013-entities.md) are.

The record will hold:

| Field | Purpose |
|---|---|
| Identifier | Names the record, and is what the cookie carries. |
| Payload | What the session holds. |
| Created, last seen | When it began, and when it was last used. |
| Client address | Stored truncated, per [ADR-0022](../adr/0022-client-addresses-are-stored-truncated-by-default.md). |
| User agent | As sent by the client. |
| Expiry | When it lapses, whether by age or by idleness. |

The address and user agent will be recorded so that a person reading the table, or a page listing their own
sessions, can tell one from another. Neither will be used to decide whether a session is valid: binding a session to
an address signs out anyone whose network changes, and binding it to a user agent breaks on a browser update.

### Identifiers

A session identifier will be 32 random bytes, hex encoded. It will not be an
[entity identifier](../GLOSSARY.md#entity-identifier): those are ULIDs, as
[ADR-0011](../adr/0011-identifiers-are-ulids-generated-in-php.md) records, which embed a timestamp and a counter and are
therefore guessable from a known one. A session identifier is a bearer credential, so it has to be unguessable.

The identifier will be regenerated whenever privilege changes, which means signing in and any elevation.

Regenerating will write a new record and mark the old one as a [superseded record](../GLOSSARY.md#superseded-record),
pointing at the new one, for a short configurable window. A request already in flight carrying the old identifier will
resolve to the new record rather than to a stale copy of the session, which is what would otherwise be written back when
that request ends. Superseded records will be removed by the same pruning that removes expired ones.

### Lifetime and loading

A session will have the `Cycle` [lifetime](../GLOSSARY.md#lifetime) from [RFC-0007](0007-binding-lifetimes.md), so it
will be resolved once per request and discarded when the [cycle](../GLOSSARY.md#cycle) closes.

Loading will be lazy, and writing conditional:

- A request that never touches the session will read nothing.
- A session nothing changed will write nothing.

When a session does write, it will write twice, at different moments. The cookie will be written on the response, during
the request, because the response is emitted before the cycle closes. The record will be written when the cycle closes.
The client already has its response by then, so persisting will cost it nothing.

Persisting at cycle close will be the [disposal](../GLOSSARY.md#disposal) described in
[RFC-0007](0007-binding-lifetimes.md).

### Concurrency

Nothing will lock a session. A hypermedia page loads several fragments at once, so requests for one session arrive
concurrently. PHP's own session handling serialises them by locking, which would make those fragments load one after
another. The last write will win instead, which is why a session will hold little and change rarely.

Flash will be the exception, and will carry a hazard: two concurrent requests can each read a flash value before either
clears it, or one can clear it before the other reads it. Flash will be written by the request that redirects and is
meant for the document request that follows, so a fragment request must not consume it.

### The session

`Session` will read, write and discard the session's data:

| Method | Effect |
|---|---|
| `get(string $key, mixed $default = null): mixed` | The value held, or the default. |
| `put(string $key, mixed $value): void` | Holds the value. |
| `forget(string $key): void` | Drops the value. |
| `flash(string $key, mixed $value): void` | Holds the value for the next request only. |
| `pull(string $key, mixed $default = null): mixed` | Returns the value and drops it. |
| `regenerate(): void` | Issues a new identifier, superseding the old record. |
| `invalidate(): void` | Removes the record and clears the cookie. |

A flash value written during one request will be readable in the next, and gone after it.

### The cookie

The cookie will carry the identifier and nothing else. It will be unavailable to scripts, sent only over a secure
connection, scoped to the whole panel, and given no domain, so it is not shared with subdomains. Its name will be
configurable.

It will be sent on same-site requests and on inbound navigation from elsewhere, but not on cross-site form submissions.
We will not use the stricter setting, which withholds it on any inbound navigation, because following a link from an
email would then arrive signed out.

### Reaching a session from longer-lived code

The middleware that writes the cookie will have the `Process` [lifetime](../GLOSSARY.md#lifetime), as
[RFC-0010](0010-http-transport.md) describes, so taking a session as a constructor dependency would resolve one on the
first request and hold it for the life of the worker.

It will take a [provider](../GLOSSARY.md#provider) instead, as [RFC-0007](0007-binding-lifetimes.md) describes, which
resolves the session against whichever [cycle](../GLOSSARY.md#cycle) is open when it is called:

```php
final class WriteSessionCookie implements Middleware
{
    public function __construct(
        #[Provide(Session::class)] private Provider $session,
    ) {}

    public function process(Request $request, Handler $next): Response
    {
        $session = $this->session->get();
        // ...
    }
}
```

The same applies to anything else that outlives a request and needs the session.

### Expiry and pruning

A session will carry an absolute expiry and an idle timeout, both configurable. A session past either will be treated as
absent, so a request carrying its identifier will start a new one.

`SessionConfig` will hold the cookie's name, the expiry and idle timeout, and the regeneration window.

Expired records, and any [superseded record](../GLOSSARY.md#superseded-record), will be removed by a scheduled task
rather than by work attached to a request, so no visitor pays for the cleanup. There is no scheduler yet, so the query
will ship with this design and be scheduled when one exists.

### Out of scope

- **Authentication.** A session will hold whatever signs someone in and give it no meaning of its own.
- **Long-lived re-authentication.** That belongs with authentication.
- **Encrypting the payload.** It never leaves the server; only the identifier does.
- **Cross-site request forgery.** The token belongs in the session, but issuing and verifying it is a separate
  component built on this one.
- **Rate limiting.**
- **The schema itself.** The table's columns and types are written with the schema builder in
  [RFC-0012](0012-postgresql-database-layer.md).

## Alternatives considered

The decision this design rests on is recorded separately, with the alternatives it rejected:

- storing a client address truncated, in
  [ADR-0022](../adr/0022-client-addresses-are-stored-truncated-by-default.md)

**A self-contained cookie.** A signed or encrypted cookie would hold the session's data with no record on the server,
saving a read per request. We rejected this because it cannot be revoked, so suspending an account leaves its sessions
working until they expire.

**Locking a session for the duration of a request.** This is what PHP's own session handling does. We rejected this
because it serialises the concurrent fragment requests a hypermedia page depends on.

**Deleting the old record when regenerating.** We rejected this because a request already in flight would then fail,
which is why the old record is kept briefly and marked superseded instead.

**Identifying sessions with the entity identifier.** We rejected this because a ULID embeds a timestamp and a counter,
so one is guessable from another, and this identifier is a bearer credential.

No other alternatives were weighed.

## Backwards compatibility

Nothing will break. The panel has no sessions today.

## Open questions

- **What tells a session that a request is a document rather than a fragment.** Only a document request should
  consume flash, and this component cannot tell the difference: it is carried in a request header and belongs to the
  hypermedia layer.

## Changelog

- 2026-10-05: Reworded in the readable house style, and the project's own vocabulary linked to the glossary. The
  design is unchanged.

## Sources

- Issue [#67], Engine - Sessions, 2026-08-22, edited the same day: sessions as their own component and why, the
  database store and the rejected self-contained cookie, the random identifier and why it is not an entity
  identifier, regeneration on privilege change with the previous record retained, cycle lifetime with lazy loading,
  the two writes at their different moments, the absence of locking and the flash hazard, the session's methods, the
  cookie's attributes and why the looser same-site setting, reaching a session through a provider, expiry and
  scheduled pruning, and every boundary listed under Out of scope. Its edit split out cross-site request forgery and
  corrected the cookie write, which the body had said happened at cycle close.
- Issue [#65], Engine - HTTP - Middleware pipeline, 2026-08-21: middleware being process lifetime and taking a
  provider for anything shorter-lived.
- The superseded record pointing at its replacement so an in-flight request resolves to the new session rather than
  a stale copy; the window being configurable and cleaned up by the same pruning; and the record carrying created
  and last-seen timestamps, a truncated client address and the user agent, for reading rather than for validation:
  first written down on 2026-09-16.

[#65]: https://github.com/thegamepanel/panel/issues/65
[#67]: https://github.com/thegamepanel/panel/issues/67
