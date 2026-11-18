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

I will add server-side sessions, keyed by a cookie and stored in the database, with flash state carrying a value from
the request that writes it to the request that follows. A session will be resolved once per
[cycle](../GLOSSARY.md#cycle), loaded only when something reads it, and written only when something changed it. The
cookie will be set on the response during the request, and the record persisted when the cycle closes.

## Motivation

A server-rendered panel needs to remember who is using it between requests, and to carry a message across the
redirect that follows a form submission. Neither is possible with the request alone.

Sessions will be a component of their own rather than part of the HTTP layer. They will own a table, a store, an
expiry policy and a pruning task, none of which is transport. What faces HTTP will be thin: reading a cookie, and
writing one onto a response.

## Proposal

### Where a session lives

A session will live in the database, and its cookie will carry nothing but an identifier. I rejected a self-contained
cookie holding the session's data: it would save a read on every request, but it cannot be revoked. Suspending someone,
or forcing them out, has to invalidate a live session, and a cookie that carries its own data stays valid until it
expires.

`SessionStore` will be the contract for loading, persisting and removing records. The database implementation will be
the only one, named as the entity stores in [RFC-0013](0013-entities.md) are.

A record will hold:

| Field | Purpose |
|---|---|
| Identifier | Names the record, and is what the cookie carries. |
| Payload | What the session holds. |
| Created, last seen | When it began, and when it was last used. |
| Client address | Stored truncated, per [ADR-0022](../adr/0022-truncated-client-addresses.md). |
| User agent | As the client sent it. |
| Expiry | When it lapses, whether by age or by idleness. |

The address and user agent will be recorded so that someone reading the table, or a page listing a person's own
sessions, can tell one session from another. Neither will decide whether a session is valid: binding a session to an
address signs out anyone whose network changes, and binding it to a user agent breaks it on a browser update.

### Identifiers

A session identifier will be 32 random bytes, hex encoded. It will not be an [entity
identifier](../GLOSSARY.md#entity-identifier). Those are ULIDs, as
[ADR-0011](../adr/0011-ulid-identifiers.md) decides, and a ULID embeds a timestamp and a counter,
so one can be guessed from another. A session identifier is a bearer credential, so it has to be unguessable.

The identifier will be regenerated whenever privilege changes: on signing in, and on any elevation. Regenerating will
write a new record and keep the old one for a short, configurable window as a [superseded
record](../GLOSSARY.md#superseded-record) pointing at the new one. A request already in flight with the old identifier
will then resolve to the new record, rather than failing or loading a stale copy of the session that it would write back
when it ends. Superseded records will be removed by the same pruning that removes expired ones.

### Loading and writing

A session will have the `Cycle` [lifetime](../GLOSSARY.md#lifetime) from [RFC-0007](0007-binding-lifetimes.md), so it
will be resolved once per request and discarded when the [cycle](../GLOSSARY.md#cycle) closes.

Loading will be lazy and writing conditional. A request that never touches the session will read nothing, and a
session that nothing changed will write nothing.

A session that does write will write twice, at different moments. The cookie will be set on the response during the
request, because the response is emitted before the cycle closes. The record will be persisted when the cycle closes, as
the [disposal](../GLOSSARY.md#disposal) that [RFC-0007](0007-binding-lifetimes.md) describes. The client already has its
response by then, so persisting costs it nothing.

### Concurrent requests

Nothing will lock a session. A hypermedia page loads several fragments at once, so requests for one session arrive
concurrently. PHP's own session handling serialises them by locking, which would load those fragments one after
another. The last write will win instead, which is why a session will hold little and change rarely.

Flash will be the exception, and carries a hazard: two concurrent requests can each read a flash value before either
clears it, or one can clear it before the other reads it. Flash will be written by the request that redirects and is
meant for the document request that follows, so a fragment request must not consume it.

### Using a session

A session will hold two types of state by key. Ordinary state will persist until something changes it: `put()` will
hold a value, `get()` will return it, and `forget()` will drop it. Flash state will live for exactly the next request:
a value held with `flash()` during one request will be readable in the next and gone after that, which is what carries
a message across the redirect that follows a form submission. `get()` will return a default when there is no value,
null unless one is given, and `pull()` will return a value, or a default when there is none, and drop it in one step.

A session will also manage its own identifier. `regenerate()` will issue a new one and supersede the old record, as
described under Identifiers, and `invalidate()` will end the session by removing its record and clearing the cookie.

### The cookie

The cookie will carry the identifier and nothing else. It will be unavailable to scripts, sent only over a secure
connection, scoped to the whole panel, and given no domain, so it is not shared with subdomains. Its name will be
configurable.

It will be sent on same-site requests and on navigation arriving from elsewhere, but not on cross-site form
submissions. I will not use the stricter setting, which withholds the cookie on any navigation arriving from another
site, because following a link from an email would then arrive signed out.

### Reaching a session from longer-lived code

The middleware that writes the cookie will have the `Process` [lifetime](../GLOSSARY.md#lifetime), as
[RFC-0010](0010-http-transport.md) describes. Taking a session as a constructor dependency would resolve one on the
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

Anything else that outlives a request and needs the session will reach it the same way.

### Expiry and pruning

A session will carry an absolute expiry and an idle timeout, both configurable. A session past either will be treated
as absent, so a request carrying its identifier will start a new one.

`SessionConfig` will hold the cookie's name, the absolute expiry, the idle timeout and the regeneration window.

Expired records, and every [superseded record](../GLOSSARY.md#superseded-record), will be removed by a scheduled task
rather than by work attached to requests, so no visitor pays for the cleanup. There is no scheduler yet, so the pruning
query will ship with this design and be scheduled once one exists.

### Out of scope

- **Authentication.** A session will hold whatever signs someone in, and give it no meaning of its own.
- **Long-lived re-authentication.** That belongs with authentication.
- **Encrypting the payload.** It never leaves the server; only the identifier does.
- **Cross-site request forgery.** Its token belongs in the session, but issuing and verifying it is a separate
  component built on this one.
- **Rate limiting.**
- **The schema itself.** The table's columns and types are written with the schema builder in
  [RFC-0012](0012-postgresql-database-layer.md).

## Alternatives considered

This design rests on [ADR-0022](../adr/0022-truncated-client-addresses.md), which stores a
client address truncated.

**A self-contained cookie.** A signed or encrypted cookie would hold the session's data with no record on the server,
saving a read on every request. I rejected this because it cannot be revoked, so suspending an account would leave its
sessions working until they expire.

**Locking a session for the duration of a request.** This is what PHP's own session handling does. I rejected this
because it serialises the concurrent fragment requests a hypermedia page depends on.

**Deleting the old record when regenerating.** I rejected this because a request already in flight would then fail,
which is why the old record is kept briefly and marked superseded instead.

**Identifying sessions with the entity identifier.** I rejected this because a ULID embeds a timestamp and a counter,
so one is guessable from another, and a session identifier is a bearer credential.

## Backwards compatibility

Nothing will break. The panel has no sessions today.

## Open questions

### What tells a session that a request is a document rather than a fragment

Only a document request should consume flash, and this component cannot tell the difference: it is carried in a
request header and belongs to the hypermedia layer.

## Changelog

- 2026-10-05: Reworded in the readable house style, and the project's own vocabulary linked to the glossary. The
  design is unchanged.
- 2026-11-18: Rewrote the document in full, with no change to the design.

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
