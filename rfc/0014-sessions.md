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

Server-side sessions, keyed by a cookie and stored in the database. A session is resolved once per cycle, loaded
only when something reads it, and written only when something changed. The cookie is set on the response during the
request and the record is persisted when the cycle closes. Flash state carries one value from the request that
writes it to the request that follows.

## Motivation

A server-rendered panel needs to remember who is using it between requests, and to carry a message across the
redirect that follows a form submission. Neither is possible with the request alone.

Sessions are their own component rather than part of the HTTP layer. They own a table, a store, an expiry policy and
a pruning task, none of which is transport. What is HTTP-facing is thin: reading a cookie, and writing one onto a
response.

## Proposal

### Concepts

| Term | Meaning |
|---|---|
| Session | The data kept for one visitor between requests, identified by a cookie. |
| Session identifier | The value in the cookie, naming a record. |
| Flash | A value written in one request, readable in the next, and gone after it. |
| Superseded record | A record replaced by regeneration, kept briefly so requests already in flight still resolve. |

### Components

| Component | Responsibility |
|---|---|
| `Session` | The session itself: reading, writing and discarding its data. |
| `SessionStore` | Contract for loading, persisting and removing records, with a database implementation. |
| `SessionConfig` | The cookie's name, the expiry and idle timeout, and the regeneration window. |

### Storage

A session lives in the database, and the cookie carries nothing but an identifier. A self-contained cookie holding
the session's data would save a read on every request, and is rejected because it cannot be revoked: suspending
someone, or forcing them out, has to invalidate a live session, and a cookie that carries its own data stays valid
until it expires.

`SessionStore` is the contract and the database implementation is the one that exists, named as the entity stores in
[RFC-0013](0013-entities.md) are.

The record holds:

| Field | Purpose |
|---|---|
| Identifier | Names the record, and is what the cookie carries. |
| Payload | What the session holds. |
| Created, last seen | When it began, and when it was last used. |
| Client address | Stored truncated, per [ADR-0022](../adr/0022-client-addresses-are-stored-truncated-by-default.md). |
| User agent | As sent by the client. |
| Expiry | When it lapses, whether by age or by idleness. |

The address and user agent are recorded so that a person reading the table, or a page listing their own sessions,
can tell one from another. Neither is used to decide whether a session is valid: binding a session to an address
signs out anyone whose network changes, and binding it to a user agent breaks on a browser update.

### Identifiers

A session identifier is 32 random bytes, hex encoded. It is not an entity identifier: those are ULIDs, per
[ADR-0011](../adr/0011-identifiers-are-ulids-generated-in-php.md), which embed a timestamp and a counter and are
therefore guessable from a known one. A session identifier is a bearer credential, so it has to be unguessable.

The identifier is regenerated whenever privilege changes, which means signing in and any elevation.

Regenerating writes a new record and marks the old one as superseded by it, for a short configurable window. A
request already in flight carrying the old identifier resolves to the new record rather than to a stale copy of the
session, which is what would otherwise be written back when that request ends. Superseded records are removed by the
same pruning that removes expired ones.

### Lifetime and loading

A session is `Cycle` lifetime, per [RFC-0007](0007-binding-lifetimes.md), so it is resolved once per request and
discarded when the cycle closes.

Loading is lazy, and writing is conditional:

- A request that never touches the session reads nothing.
- A session nothing changed writes nothing.

When it does write, it writes twice, at different moments:

| What | When | Why |
|---|---|---|
| The cookie | On the response, during the request | The response is emitted before the cycle closes. |
| The record | When the cycle closes | The client already has its response by then, so persisting costs it nothing. |

Persisting at cycle close is the disposal described in [RFC-0007](0007-binding-lifetimes.md).

### Concurrency

Nothing locks a session. A hypermedia page loads several fragments at once, so requests arrive concurrently for one
session, and PHP's own session handling serialises them by locking, which would make those fragments load one after
another. The last write wins instead, which is why a session holds little and changes rarely.

Flash is the exception, and carries a hazard worth stating: two concurrent requests can each read a flash value
before either clears it, or one can clear it before the other reads it. Flash is written by the request that
redirects and meant for the document request that follows, so a fragment request must not consume it.

### The session

| Method | Effect |
|---|---|
| `get(string $key, mixed $default = null): mixed` | The value held, or the default. |
| `put(string $key, mixed $value): void` | Holds the value. |
| `forget(string $key): void` | Drops the value. |
| `flash(string $key, mixed $value): void` | Holds the value for the next request only. |
| `pull(string $key, mixed $default = null): mixed` | Returns the value and drops it. |
| `regenerate(): void` | Issues a new identifier, superseding the old record. |
| `invalidate(): void` | Removes the record and clears the cookie. |

A flash value written during one request is readable in the next, and gone after it.

### The cookie

The cookie carries the identifier and nothing else. It is unavailable to scripts, sent only over a secure
connection, scoped to the whole panel, and given no domain, so it is not shared with subdomains. Its name is
configurable.

It is sent on same-site requests and on inbound navigation from elsewhere, but not on cross-site form submissions.
The stricter setting, which withholds it on any inbound navigation, would mean following a link from an email
arrives signed out.

### Reaching a session from longer-lived code

The middleware that writes the cookie is `Process` lifetime, per [RFC-0010](0010-http-transport.md), so taking a
session as a constructor dependency would resolve one on the first request and hold it for the life of the worker.

It takes a provider instead, per [RFC-0007](0007-binding-lifetimes.md), which resolves the session against whichever
cycle is open when it is called:

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

A session carries an absolute expiry and an idle timeout, both configurable. A session past either is treated as
absent, so a request carrying its identifier starts a new one.

Expired and superseded records are removed by a scheduled task rather than by work attached to a request, so no
visitor pays for the cleanup. There is no scheduler yet, so the query ships with this design and is scheduled when
one exists.

### Out of scope

- **Authentication.** A session holds whatever signs someone in and gives it no meaning of its own.
- **Long-lived re-authentication**, which belongs with authentication.
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

**A self-contained cookie**, signed or encrypted, holding the session's data with no record on the server. It saves
a read per request and cannot be revoked, so suspending an account leaves its sessions working until they expire.

**Locking a session for the duration of a request**, as PHP's own handling does. It serialises the concurrent
fragment requests a hypermedia page depends on.

**Deleting the old record when regenerating.** A request already in flight then fails, which is why the old record
is kept briefly and marked superseded instead.

**Identifying sessions with the entity identifier.** A ULID embeds a timestamp and a counter, so one is guessable
from another, and this identifier is a bearer credential.

No other alternatives were weighed.

## Backwards compatibility

Nothing breaks. The panel has no sessions before this.

## Open questions

- **What tells a session that a request is a document rather than a fragment.** Only a document request should
  consume flash, and this component cannot tell the difference: it is carried in a request header and belongs to the
  hypermedia layer.

## Changelog

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
