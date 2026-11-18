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

I will build the [engine](../GLOSSARY.md#engine)'s HTTP layer from immutable request and response objects, a middleware
pipeline wrapped around a terminal handler, and a route table the engine owns, which requests are matched against and
links are generated from. The layer will hydrate a request from the superglobals, run it through the pipeline, resolve a
handler and emit the response. It will stop at the handler seam.

## Motivation

The [engine](../GLOSSARY.md#engine) has no HTTP layer. `nikic/fast-route`, `filp/whoops` and `monolog/monolog` have been
required since the start and are unused.

The panel is server-rendered, as [ADR-0001](../adr/0001-single-system.md)
decides, so the engine has to accept a request, decide what answers it, and produce HTML. Server rendering also means
every anchor and every form action is generated on the server, so routes have to run backwards as well as forwards.

## Proposal

### The request

The messages will be bespoke rather than PSR-7, as [ADR-0018](../adr/0018-bespoke-http-messages.md)
decides. A `Request` will describe what arrived. It will be constructed once per [cycle](../GLOSSARY.md#cycle) and never
changed, and it will carry:

- the method, as the `Method` enum;
- the URI, as a `Uri` holding the scheme, host, port, path and query string, with the scheme resolved through the
  proxy headers;
- the headers, as `Headers`, taken from the `HTTP_*` server values plus the content type and length;
- the cookies, as `Cookies`;
- the query string values and the decoded form body, each as an array;
- the uploaded files, as `UploadedFiles`, which normalises PHP's nested shape into `UploadedFile` values;
- the raw body, as a stream;
- the client address, as an `IpAddress`.

`Method` will cover GET, HEAD, POST, PUT, PATCH, DELETE and OPTIONS. A method outside that set will be a routing
outcome, not a failure to hydrate the request.

`Headers` will be immutable, with case-insensitive lookup and any number of values for each name. It will be normalised
from the server values rather than from `getallheaders()`, so its source is the same under a test as under the worker.

### Reading the request from the worker

`Request::fromGlobals()` will build a request from the superglobals, which FrankenPHP, the runtime
[ADR-0009](../adr/0009-frankenphp-worker.md) chooses, repopulates for each
request inside the worker callback. A general constructor will take the same values explicitly, so a test can build a
request without touching global state.

By the time any [engine](../GLOSSARY.md#engine) code runs, PHP has already parsed the form body, decoded its encoding,
folded the headers and populated the uploaded files. The request will therefore hydrate rather than parse. The one thing
left raw is the body, which will not be decoded here. It will be read once and held, and it will stay a stream, because
a backup upload can run to gigabytes. When the request was multipart, PHP has already consumed the body into the form
values and the uploaded files, and it cannot be read again.

### The client address

The reverse proxy in front of the worker is always the immediate peer, so the connection's own address is the
proxy's. The client address will come from the forwarded-for header instead, trusting loopback by default, with the set
of trusted proxies configurable.

`IpAddress` will normalise and validate both IPv4 and IPv6 addresses. It will live with the
[engine](../GLOSSARY.md#engine)'s shared values rather than in the HTTP layer, because tokens and audit records use it
too.

### No attribute bag

A request will carry no general-purpose bag of properties, and nothing will copy a request in order to attach something
to it. Middleware that works something out, such as the authenticated user or the matched route, will bind it into the
open [cycle](../GLOSSARY.md#cycle) from [RFC-0007](0007-binding-lifetimes.md), and whatever needs it will resolve it
from the container. A request will stay a description of what arrived.

### Responses

A `Response` will carry a status, headers, cookies and a body. It will take no position on content type: that will be a
header, set by whatever produced the body. Each cookie it sets will be a `Cookie`, carrying its name, value, expiry,
path, domain, and its secure, HTTP-only and same-site attributes.

Responses will be built through static factories:

```php
Response::make(string $body, Status $status = Status::Ok): Response;
Response::redirect(string $location, Status $status = Status::Found): Response;
Response::noContent(): Response;
Response::stream(Closure $writer, Status $status = Status::Ok): Response;
```

`make()` will cover a body of any type. Each of the other three will exist because it sets something beyond the body: a
redirect sets the location header, `noContent()` produces a 204 with no body, and `stream()` takes a closure in place of
a string.

`Status` will be an integer-backed enum over the registered status codes, carrying each code's reason phrase. As an
enum, it will not be able to express a code outside the registry, and non-standard codes will be left out. The reason
phrase is not sent over HTTP/2 or HTTP/3, which is what the proxy negotiates for most traffic, so it will exist for logs
and error pages.

### Streaming a body

A body will be a string, or a closure that is given a `BodyWriter` for a streamed one. File and backup downloads need
it, as will server-sent events for the console.

```php
Response::stream(static function (BodyWriter $out) use ($path): void {
    $out->file($path);
});
```

The closure will be handed a writer rather than echoing, so streaming can be tested against a fake writer instead of
through output buffering. `BodyWriter` will write a string, write a file, and flush.

`Emitter` will send a response: the status line, then the headers, the cookies and the body. A string body will be
written in one call, and a closure body will be invoked with a real writer.

### Handlers and middleware

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

`Handler` will be the seam this design stops at. Everything that turns a request into a response will implement it,
including a composed pipeline, so a pipeline will be substitutable for the handler it wraps.

Middleware will be handed the next handler rather than a closure, so a test can give a middleware the terminal handler
directly. A middleware that returns without calling it will short-circuit everything inside it: an authentication
middleware returning a redirect will never reach what it wraps.

```php
$handler = Pipeline::through($middleware)->then($terminal);
```

Middleware will be given as class names and resolved through the container, as
[RFC-0001](0001-dependency-injection-container.md) describes, when the pipeline is composed rather than on every
request. They will have the `Process` [lifetime](../GLOSSARY.md#lifetime) from [RFC-0007](0007-binding-lifetimes.md), so
one instance will serve every request the worker handles. A middleware will hold no state belonging to a request, and
will reach anything that does through a [provider](../GLOSSARY.md#provider).

### When pipelines are composed

Pipelines will be composed at two moments. The outer pipeline, holding the middleware every request passes through
whatever it matches, will wrap the router and be composed once at boot.

A route's own pipeline cannot be composed at boot, because which middleware apply depends on the route matched. It will
be composed the first time its route is used instead, and kept for the life of the worker, keyed by route, so the cost
is paid once for each route in each worker. Composing on every request is what a runtime that boots for each request
forces, and a worker does not.

### Middleware order

Middleware will run outward to inward and unwind in reverse. They will come from three sources, applied in order:
global, then group, then route, each in the order it was declared. A middleware named by more than one source will run
once, at its earliest position.

There will be no priority numbers. Order will be positional, so a middleware that must run before another says so by
where it appears in the list.

### The route table

FastRoute matches a path against patterns and answers found, not found or method not allowed. It does not hold routes
in a form anything else can read, and it cannot run backwards. Since every anchor and form action is generated, the
[engine](../GLOSSARY.md#engine) will own the route table, and compile FastRoute's dispatch data from it.

Routes will be collected in a `RouteRegistry` and sealed into a `RouteCatalogue`, following
[ADR-0003](../adr/0003-sealed-catalogues.md). The
[catalogue](../GLOSSARY.md#catalogue) will be indexed by name, and will hold the dispatch data along with each pattern
parsed into its variants, which is what generation needs. Both will be built once at boot and held for the life of the
worker, so no compiled form will be cached to a file.

A `Route` will carry the methods it answers, its path pattern, its name, the handler behind it, its middleware and its
[panel context](../GLOSSARY.md#panel-context). Routes will be a projection of the action registry, so every route will
have an action behind it, its name will come from that action, and nothing will declare a route name by hand.

### Matching a request

The router will be a `Handler`. It will match the request, then delegate:

| Outcome | Result |
|---|---|
| Found | The route's pipeline runs, wrapping the handler behind the route. |
| Not found | A 404 status, with its presentation left to whatever renders errors. |
| Method not allowed | A 405 status, with an allow header listing the methods the path accepts. |

`HEAD` will be served by the matching `GET` route, with the body suppressed when the response is emitted. Paths will be
case sensitive, and a path that differs from its canonical form only by a trailing slash will redirect to the canonical
form rather than match. Parameters will be extracted as strings; turning one into an entity belongs to whatever
consumes the route.

Each route will belong to a [panel context](../GLOSSARY.md#panel-context), and each context will map to a URL prefix.
The prefixes will come from configuration rather than code, since the panel decides its own URL shape.

### Generating links

Generation will not be a service. The [catalogue](../GLOSSARY.md#catalogue) will look a route up by name, and the route
will substitute parameters into its own pattern, validating them against its own constraints. A missing or invalid
parameter will throw rather than produce a broken link.

```php
$catalogue->get('server.show')->path(['server' => $id]);
```

Absolute URLs will not be built here. The scheme and host will come from configuration rather than from the current
request, so a link generated in a queued job will match one generated in a request, and whatever needs an absolute URL
will combine the two.

### Errors

A middleware may catch what it wraps, but the pipeline will catch nothing, and neither will the router. Turning an
exception into a response belongs to whatever handles errors, and containing one belongs to whatever drives the worker.

### Out of scope

- **The worker loop.** Booting the framework and running it, with drivers for HTTP, the command line and the queue,
  belongs to bootstrapping. Anything serving every driver is not an HTTP concern.
- **Errors and diagnostics.** Turning exceptions into responses, logging and error pages serve every driver too.
- **Views.** Templates, layouts and fragment rendering produce the body this layer emits.
- **Actions.** The [module](../GLOSSARY.md#module)-facing layer sits on the other side of the handler seam, and owns how
  HTTP methods are used, how parameters become entities, and what a result maps to.
- **Sessions.** They own a table, a store and an expiry policy. Their cookies are transported here.
- **Route authoring.** Routes arrive from the action registry; this design defines the shape they arrive in.
- **Body decoding and content types.** Decoding a body and deciding a content type belong to what consumes and
  produces it.
- **Cross-origin handling, method override, and built-in middleware.** There is no cross-origin consumer, how methods
  are used is decided where actions are defined, and the middleware that exist arrive with what needs them.
- **Modules contributing middleware or routes**, which is a [collector](../GLOSSARY.md#collector) in the module system.

## Alternatives considered

This design rests on four decisions, each recorded with the alternatives it rejected:

- [ADR-0018](../adr/0018-bespoke-http-messages.md), bespoke messages rather than PSR-7 and PSR-15;
- [ADR-0001](../adr/0001-single-system.md), the panel being one
  server-rendered system;
- [ADR-0003](../adr/0003-sealed-catalogues.md), sealing mutable registries into
  immutable catalogues;
- [ADR-0009](../adr/0009-frankenphp-worker.md), running as a worker.

**Using FastRoute's route collection as the route table.** I rejected this because it can neither be read back nor run
backwards, and generation needs both.

**FastRoute's file-backed cached dispatcher.** I rejected this because the worker already holds the compiled form in
memory, so a file would add a staleness problem for no gain.

**Composing every pipeline at boot.** I rejected this because which middleware apply to a route depends on the route
matched.

## Backwards compatibility

Nothing will break. The [engine](../GLOSSARY.md#engine) has no HTTP layer today.

## Open questions

## Resolved questions

### Where the panel context is defined

Routing needs it, and it is currently defined inside the module system's collector work. It has to be promoted
somewhere shared before routing can consume it.

> Resolved by [RFC-0015](0015-module-system.md), which defines it in the [engine](../GLOSSARY.md#engine)'s shared
> values, alongside the client address.

## Changelog

- 2026-09-16: Where the panel context is defined is resolved by [RFC-0015](0015-module-system.md).
- 2026-10-05: Reworded in the readable house style, and the project's own vocabulary linked to the glossary. The
  design is unchanged.
- 2026-11-18: Rewrote the document in full, with no change to the design.
- 2026-11-18: Moved where the panel context is defined to Resolved questions, answered by
  [RFC-0015](0015-module-system.md).

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
