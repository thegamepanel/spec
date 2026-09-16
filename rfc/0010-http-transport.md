---
id: RFC-0010
title: HTTP transport
status: proposed
created: 2026-09-16
decided:
depends: [ADR-0003, ADR-0009, ADR-0018, RFC-0001, RFC-0007]
updates: []
obsoletes: []
---

# RFC-0010: HTTP transport

## Abstract

The engine's HTTP layer: immutable request and response objects hydrated from the superglobals and emitted back, a
middleware pipeline wrapping a terminal handler, and a route table the engine owns, matched against and generated
from. It accepts a request, runs it through the pipeline, resolves a handler and emits a response, and stops at the
handler seam.

## Motivation

The engine has no HTTP layer. `nikic/fast-route`, `filp/whoops` and `monolog/monolog` have been required since the
start and are unused.

The panel is server-rendered, per
[ADR-0001](../adr/0001-the-panel-is-a-single-system-not-a-panel-and-an-engine.md), so the engine has to accept a
request, decide what answers it, and produce HTML. Server-rendering also means every anchor and every form action is
generated on the server, so routes have to run backwards as well as forwards.

## Proposal

### Concepts

| Term | Meaning |
|---|---|
| Request | An immutable description of what arrived over the wire. |
| Response | Status, headers, cookies and a body, ready to be emitted. |
| Handler | Anything that turns a request into a response. |
| Middleware | Something wrapped around a handler, which may act before it, after it, or instead of it. |
| Pipeline | Middleware composed around a terminal handler, itself a handler. |
| Route | One path pattern, the methods it answers, and what handles it. |
| Panel context | The part of the panel a route belongs to, which decides its URL prefix. |

### Components

| Component | Responsibility |
|---|---|
| `Request`, `Response` | The messages, bespoke rather than PSR-7, per [ADR-0018](../adr/0018-http-messages-are-bespoke-not-psr-7.md). |
| `Headers`, `Cookies`, `Cookie`, `Uri`, `UploadedFiles`, `UploadedFile` | The collections and values a message carries. |
| `Method`, `Status` | Enums for the request method and the response status. |
| `BodyWriter`, `Emitter` | Writing a streamed body, and emitting a response. |
| `Handler`, `Middleware`, `Pipeline` | The contracts for handling a request, and their composition. |
| `Route`, `RouteRegistry`, `RouteCatalogue` | The route table, collected and then sealed. |
| `Router` | A handler that matches a request to a route and delegates to it. |

### Messages

A `Request` is constructed once per cycle and never changed.

| Member | Type | Taken from |
|---|---|---|
| `method` | `Method` | The request method. |
| `uri` | `Uri` | The server values, with the scheme resolved through the proxy headers. |
| `headers` | `Headers` | The `HTTP_*` server values, plus the content type and length. |
| `cookies` | `Cookies` | The request cookies. |
| `query` | `array` | The query string values. |
| `form` | `array` | The decoded form body. |
| `files` | `UploadedFiles` | The uploaded files, with the nested shape normalised. |
| `body` | Stream | The raw body. |
| `ip` | `IpAddress` | The client address, resolved through trusted proxies. |

`Request::fromGlobals()` hydrates from the superglobals, which FrankenPHP repopulates for each request inside the
worker callback, per [ADR-0009](../adr/0009-the-panel-runs-as-a-frankenphp-worker-in-a-single-binary.md). A general
constructor takes the same values explicitly, so a test builds a request without touching global state.

PHP has already parsed the form body, decoded the encoding, folded the headers and populated the uploaded files
before any engine code runs, so the request hydrates rather than parses. The body is the raw stream, read once and
held, and is not decoded here: it stays a stream because uploads are measured in gigabytes. When the request was
multipart, PHP has already consumed it.

Headers are normalised from the server values rather than from `getallheaders()`, so the source is the same under a
test as under the worker. Lookup is case insensitive and values are multi-valued.

`Method` covers GET, HEAD, POST, PUT, PATCH, DELETE and OPTIONS. A method outside that set is a routing outcome, not
a hydration failure.

The client address comes from the forwarded-for header, since the reverse proxy in front of the worker is always the
immediate peer. Loopback is trusted by default and the trusted set is configurable. `IpAddress` normalises and
validates both address families, and belongs in the engine's shared values rather than here, because tokens and
audit records use it too.

#### No attribute bag

A request carries no general-purpose property bag, and nothing copies a request to attach something to it. Middleware
that resolves something, such as the authenticated user or the matched route, binds it into the open cycle from
[RFC-0007](0007-binding-lifetimes.md), and whatever needs it resolves it from the container. A request stays a
description of what arrived.

#### Responses

`Response` carries a status, headers, cookies and a body, and takes no position on content type: the content type is
a header, set by whatever produced the body.

| Factory | Produces |
|---|---|
| `make(string $body, Status $status = Status::Ok)` | A response with a body, whatever its type. |
| `redirect(string $location, Status $status = Status::Found)` | A redirect, setting the location header. |
| `noContent()` | A 204 with no body. |
| `stream(Closure $writer, Status $status = Status::Ok)` | A streamed body, written by the closure. |

`Status` is an integer-backed enum over the registered status codes, carrying the reason phrase. Being an enum, a
code outside the registry cannot be expressed, and the non-standard codes are left out. The reason phrase is not
transmitted over HTTP/2 or HTTP/3, which is what the proxy negotiates for most traffic; it exists for logs and error
pages.

A body is a string, or a closure taking a `BodyWriter` for a streamed one. The closure is given a writer rather than
writing directly, so streaming is tested against a fake writer instead of output buffering. `BodyWriter` writes a
string, writes a file and flushes.

```php
Response::stream(static function (BodyWriter $out) use ($path): void {
    $out->file($path);
});
```

`Emitter` writes a response: the status line, the headers, the cookies, then the body, whether that body is a string
or a closure.

### The pipeline

```php
interface Handler
{
    public function handle(Request $request): Response;
}

interface Middleware
{
    public function process(Request $request, Handler $next): Response;
}
```

`Handler` is the seam this design stops at. Everything that turns a request into a response implements it, including
a composed pipeline, so a pipeline is substitutable for the handler it wraps.

Middleware receives the next handler rather than a closure, so a test hands a middleware the terminal handler
directly. Returning without calling it short-circuits: an authentication middleware returning a redirect never
reaches what it wrapped.

```php
$handler = Pipeline::through($middleware)->then($terminal);
```

Middleware are given as class names and resolved through the container, per
[RFC-0001](0001-dependency-injection-container.md), when the pipeline is composed rather than on every request. They
are `Process` lifetime, per [RFC-0007](0007-binding-lifetimes.md), so a middleware holds no state belonging to a
request and reaches anything that does through a provider.

The pipeline is composed at two moments. The outer pipeline wraps the router and is composed once at boot, since
every request passes through it whatever it matches. A route's pipeline cannot be composed at boot, because which
middleware apply depends on the route matched, so it is composed on first use and kept for the life of the worker,
keyed by route. Composing on every request is what a runtime that boots per request forces and a worker does not.

Middleware run outward to inward and unwind in reverse. Three sources apply in order, global, then group, then route,
and declaration order within each. A middleware named by more than one source runs once, at its earliest position.
There are no priority numbers: order is positional.

### Routing

The route table belongs to the engine. FastRoute matches a path against patterns and answers found, not found or
method not allowed; it neither holds routes in a form anything else can read nor runs backwards. Since every anchor
and form action is generated, the engine owns the table and compiles FastRoute from it.

`RouteRegistry` collects routes and seals into `RouteCatalogue`, per
[ADR-0003](../adr/0003-mutable-registries-are-sealed-into-immutable-catalogues.md). The catalogue is indexed by name
and holds the dispatch data compiled from the table, along with each pattern parsed into its variants, which is what
generation needs. Both are built once at boot and held for the life of the worker, so no compiled form is cached to
a file.

A `Route` carries the methods it answers, its path pattern, its name, the handler behind it, its middleware and its
panel context. Routes are a projection of the action registry, so a route's name comes from the action behind it and
nothing declares a name by hand.

The router is a `Handler`. It matches, then delegates:

| Outcome | Result |
|---|---|
| Found | The route's pipeline runs, wrapping the handler behind the route. |
| Not found | A 404 status, with presentation left to whatever renders errors. |
| Method not allowed | A 405 status carrying the permitted methods in an allow header. |

`HEAD` is served by the matching `GET` route with the body suppressed when the response is emitted. Paths are case
sensitive. A path differing from its canonical form only by a trailing slash redirects to the canonical form.
Parameters are extracted as strings; turning one into an entity belongs to whatever consumes the route.

Each route belongs to a panel context, and each context maps to a URL prefix taken from configuration rather than
fixed in code, since the panel decides its own URL shape.

Generation is not a service. The catalogue looks a route up by name, and the route substitutes parameters into its
own pattern, validating them against its own constraints, so a missing or invalid parameter throws rather than
producing a broken link.

```php
$catalogue->get('server.show')->path(['server' => $id]);
```

An absolute URL is not built here. The scheme and host come from configuration rather than from the current request,
so a link generated in a queued job matches one generated in a request, and whatever needs an absolute URL combines
the two.

### Errors

A middleware may catch what it wraps, but the pipeline catches nothing, and neither does the router. Turning an
exception into a response belongs to whatever handles errors, and containing one belongs to whatever drives the
worker.

### Out of scope

- **The worker loop.** Booting the framework and running it, with drivers for HTTP, the command line and the queue,
  belongs to bootstrapping. Anything serving every driver is not an HTTP concern.
- **Errors and diagnostics.** Turning exceptions into responses, logging and error pages serve every driver too.
- **Views.** Templates, layouts and fragment rendering produce the body this layer emits.
- **Actions.** The module-facing layer sits on the other side of the handler seam, and owns how methods are used,
  how parameters become entities, and what a result maps to.
- **Sessions.** They own a table, a store and an expiry policy. Their cookies are transported here.
- **Route authoring.** Routes arrive from the action registry; this defines the shape they arrive in.
- **Body decoding and content types.** Decoding a body and deciding a content type belong to what consumes and
  produces it.
- **Cross-origin handling, method override, and built-in middleware.** There is no cross-origin consumer, how methods
  are used is decided where actions are defined, and the middleware that exist arrive with what needs them.
- **Modules contributing middleware or routes**, which is a collector in the module system.

## Alternatives considered

The decisions this design rests on are recorded separately, with the alternatives each one rejected:

- bespoke messages rather than PSR-7 and PSR-15, in [ADR-0018](../adr/0018-http-messages-are-bespoke-not-psr-7.md)
- the panel being one server-rendered system, in
  [ADR-0001](../adr/0001-the-panel-is-a-single-system-not-a-panel-and-an-engine.md)
- running as a worker, in [ADR-0009](../adr/0009-the-panel-runs-as-a-frankenphp-worker-in-a-single-binary.md)

**Using FastRoute's route collection as the route table.** It cannot be read back or run backwards, and generation
needs both, so the table is the engine's and FastRoute is compiled from it.

**FastRoute's file-backed cached dispatcher.** The worker already holds the compiled form in memory, so a file would
add staleness for no gain.

**Composing every pipeline at boot.** Which middleware apply to a route depends on the route matched, so a route's
pipeline is composed on first use and kept instead.

No other alternatives were weighed.

## Backwards compatibility

Nothing breaks. The engine has no HTTP layer before this.

## Open questions

- ~~**Where the panel context is defined.** Routing needs it, and it is currently defined inside the module system's
  collector work. It has to be promoted somewhere shared before routing can consume it.~~ Resolved by
  [RFC-0015](0015-module-system.md), which defines it in the engine's shared values, alongside the client address.

## Changelog

- 2026-09-16: Where the panel context is defined is resolved by [RFC-0015](0015-module-system.md).

## Sources

- Issue [#62], Engine - HTTP, 2026-08-21, edited through 2026-08-22 and with four comments: the scope stopping at the
  handler seam, the runtime it assumes, bespoke messages, the server-rendered frontend, the route table being owned
  rather than borrowed from FastRoute, what each excluded epic is, and the panel context remaining unsettled. Its
  edits narrow the scope, moving the worker loop and error handling to bootstrapping and sessions to their own
  component, and replace citations of closed issues with the content stated directly.
- Issue [#64], Engine - HTTP - Messages, 2026-08-21, edited on 2026-08-22 to state content directly: every member of
  a request and where it comes from, hydration from the superglobals and an explicit constructor, the raw body, the
  absence of an attribute bag, client address resolution, the response factories, the status enum, streamed bodies
  through a writer, the collections, and emission.
- Issue [#65], Engine - HTTP - Middleware pipeline, 2026-08-21, edited on 2026-08-22, with a comment removing module
  contribution: the handler and middleware contracts, composition, short-circuiting, the two composition points and
  the route pipeline kept for the life of the worker, ordering across the three sources with earliest-position
  deduplication and no priorities, and `Process` lifetime.
- Issue [#66], Engine - HTTP - Routing, 2026-08-21, edited on 2026-08-22 to state content directly: the owned route
  table, the registry sealed into a catalogue indexed by name, what a route carries, routes as a projection of the
  action registry, the three matching outcomes, `HEAD`, case sensitivity, trailing-slash redirection, parameters as
  strings, contexts and their configured prefixes, generation from the route itself, absolute URLs left to their
  consumer, and compilation once at boot.

[#62]: https://github.com/thegamepanel/panel/issues/62
[#64]: https://github.com/thegamepanel/panel/issues/64
[#65]: https://github.com/thegamepanel/panel/issues/65
[#66]: https://github.com/thegamepanel/panel/issues/66
