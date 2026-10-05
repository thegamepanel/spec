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

We will build the [engine](../GLOSSARY.md#engine)'s HTTP layer from immutable request and response objects, a middleware
pipeline wrapping a terminal handler, and a route table the engine owns, which requests are matched against and links
generated from. The layer will hydrate a request from the superglobals, run it through the pipeline, resolve a handler
and emit the response. It stops at the handler seam.

## Motivation

The [engine](../GLOSSARY.md#engine) has no HTTP layer. `nikic/fast-route`, `filp/whoops` and `monolog/monolog` have been
required since the start and are unused.

The panel is server-rendered, as
[ADR-0001](../adr/0001-the-panel-is-a-single-system-not-a-panel-and-an-engine.md) decides, so the engine has to accept
a request, decide what answers it, and produce HTML. Server rendering also means every anchor and every form action is
generated on the server, so routes have to run backwards as well as forwards.

## Proposal

The messages will be `Request` and `Response`, bespoke rather than PSR-7, as
[ADR-0018](../adr/0018-http-messages-are-bespoke-not-psr-7.md) decides. They will carry their collections and values as
`Headers`, `Cookies`, `Cookie`, `Uri`, `UploadedFiles` and `UploadedFile`, and the request method and response status as
the enums `Method` and `Status`. `BodyWriter` will write a streamed body, and `Emitter` will emit a response. `Handler`
and `Middleware` will be the contracts for handling a request, and `Pipeline` will compose them. Routes will be
collected by `RouteRegistry` and sealed into `RouteCatalogue`, and `Router` will be a handler that matches a request to
a `Route` and delegates to it.

### Messages

A `Request` will be constructed once per [cycle](../GLOSSARY.md#cycle) and never changed.

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

`Request::fromGlobals()` will hydrate from the superglobals, which FrankenPHP, the runtime chosen in
[ADR-0009](../adr/0009-the-panel-runs-as-a-frankenphp-worker-in-a-single-binary.md), repopulates for each request inside
the worker callback. A general constructor will take the same values explicitly, so a test can build a request without
touching global state.

PHP has already parsed the form body, decoded the encoding, folded the headers and populated the uploaded files before
any [engine](../GLOSSARY.md#engine) code runs, so the request will hydrate rather than parse. The body will be the raw
stream, read once and held, and not decoded here: it will stay a stream because uploads are measured in gigabytes. When
the request was multipart, PHP has already consumed it.

Headers will be normalised from the server values rather than from `getallheaders()`, so the source is the same under a
test as under the worker. Lookup will be case insensitive, and values will be multi-valued.

`Method` will cover GET, HEAD, POST, PUT, PATCH, DELETE and OPTIONS. A method outside that set will be a routing
outcome, not a hydration failure.

The client address will come from the forwarded-for header, since the reverse proxy in front of the worker is always the
immediate peer. Loopback will be trusted by default, and the trusted set will be configurable. `IpAddress` will
normalise and validate both address families. It will belong with the engine's shared values rather than here, because
tokens and audit records use it too.

#### No attribute bag

A request will carry no general-purpose property bag, and nothing will copy a request to attach something to it.
Middleware that resolves something, such as the authenticated user or the matched route, will bind it into the open
[cycle](../GLOSSARY.md#cycle) from [RFC-0007](0007-binding-lifetimes.md), and whatever needs it will resolve it from the
container. A request will stay a description of what arrived.

#### Responses

`Response` will carry a status, headers, cookies and a body, and take no position on content type: the content type will
be a header, set by whatever produced the body.

| Factory | Produces |
|---|---|
| `make(string $body, Status $status = Status::Ok)` | A response with a body, whatever its type. |
| `redirect(string $location, Status $status = Status::Found)` | A redirect, setting the location header. |
| `noContent()` | A 204 with no body. |
| `stream(Closure $writer, Status $status = Status::Ok)` | A streamed body, written by the closure. |

`Status` will be an integer-backed enum over the registered status codes, carrying the reason phrase. As an enum, it
will not be able to express a code outside the registry, and the non-standard codes will be left out. The reason phrase
is not transmitted over HTTP/2 or HTTP/3, which is what the proxy negotiates for most traffic; it will exist for logs
and error pages.

A body will be a string, or a closure taking a `BodyWriter` for a streamed one. The closure will be given a writer
rather than writing directly, so streaming can be tested against a fake writer instead of output buffering. `BodyWriter`
will write a string, write a file and flush.

```php
Response::stream(static function (BodyWriter $out) use ($path): void {
    $out->file($path);
});
```

`Emitter` will write a response: the status line, the headers, the cookies, then the body, whether that body is a
string or a closure.

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

`Handler` will be the seam this design stops at. Everything that turns a request into a response will implement it,
including a composed pipeline, so a pipeline will be substitutable for the handler it wraps.

Middleware will receive the next handler rather than a closure, so a test can hand a middleware the terminal handler
directly. Returning without calling it will short-circuit: an authentication middleware returning a redirect will never
reach what it wrapped.

```php
$handler = Pipeline::through($middleware)->then($terminal);
```

Middleware will be given as class names and resolved through the container, as
[RFC-0001](0001-dependency-injection-container.md) describes, when the pipeline is composed rather than on every
request. They will have the `Process` [lifetime](../GLOSSARY.md#lifetime) from [RFC-0007](0007-binding-lifetimes.md), so
a middleware will hold no state belonging to a request and will reach anything that does through a
[provider](../GLOSSARY.md#provider).

The pipeline will be composed at two moments. The outer pipeline will wrap the router and be composed once at boot,
since every request passes through it whatever it matches. A route's pipeline cannot be composed at boot, because which
middleware apply depends on the route matched. It will be composed on first use instead, and kept for the life of the
worker, keyed by route. A runtime that boots per request forces composition on every request; a worker does not.

Middleware will run outward to inward and unwind in reverse. Three sources will apply in order: global, then group, then
route, with declaration order within each. A middleware named by more than one source will run once, at its earliest
position. There will be no priority numbers: order will be positional.

### Routing

FastRoute matches a path against patterns and answers found, not found or method not allowed; it neither holds routes in
a form anything else can read nor runs backwards. Since every anchor and form action is generated, the
[engine](../GLOSSARY.md#engine) will own the route table and compile FastRoute from it.

`RouteRegistry` will collect routes and seal into `RouteCatalogue`, following
[ADR-0003](../adr/0003-mutable-registries-are-sealed-into-immutable-catalogues.md). The
[catalogue](../GLOSSARY.md#catalogue) will be indexed by name and hold the dispatch data compiled from the table, along
with each pattern parsed into its variants, which is what generation needs. Both will be built once at boot and held for
the life of the worker, so no compiled form will be cached to a file.

A `Route` will carry the methods it answers, its path pattern, its name, the handler behind it, its middleware and its
[panel context](../GLOSSARY.md#panel-context). Routes will be a projection of the action registry, so a route's name
will come from the action behind it and nothing will declare a name by hand.

The router will be a `Handler`. It will match, then delegate:

| Outcome | Result |
|---|---|
| Found | The route's pipeline runs, wrapping the handler behind the route. |
| Not found | A 404 status, with presentation left to whatever renders errors. |
| Method not allowed | A 405 status carrying the permitted methods in an allow header. |

`HEAD` will be served by the matching `GET` route, with the body suppressed when the response is emitted. Paths will be
case sensitive. A path differing from its canonical form only by a trailing slash will redirect to the canonical form.
Parameters will be extracted as strings; turning one into an entity belongs to whatever consumes the route.

Each route will belong to a panel context, and each context will map to a URL prefix taken from configuration rather
than fixed in code, since the panel decides its own URL shape.

Generation will not be a service. The catalogue will look a route up by name. The route will substitute parameters into
its own pattern and validate them against its own constraints, so a missing or invalid parameter will throw rather than
produce a broken link.

```php
$catalogue->get('server.show')->path(['server' => $id]);
```

An absolute URL will not be built here. The scheme and host will come from configuration rather than from the current
request, so a link generated in a queued job will match one generated in a request, and whatever needs an absolute URL
will combine the two.

### Errors

A middleware may catch what it wraps, but the pipeline will catch nothing, and neither will the router. Turning an
exception into a response belongs to whatever handles errors, and containing one belongs to whatever drives the
worker.

### Out of scope

- **The worker loop.** Booting the framework and running it, with drivers for HTTP, the command line and the queue,
  belongs to bootstrapping. Anything serving every driver is not an HTTP concern.
- **Errors and diagnostics.** Turning exceptions into responses, logging and error pages serve every driver too.
- **Views.** Templates, layouts and fragment rendering produce the body this layer emits.
- **Actions.** The [module](../GLOSSARY.md#module)-facing layer sits on the other side of the handler seam, and owns how
  methods are used, how parameters become entities, and what a result maps to.
- **Sessions.** They own a table, a store and an expiry policy. Their cookies are transported here.
- **Route authoring.** Routes arrive from the action registry; this design defines the shape they arrive in.
- **Body decoding and content types.** Decoding a body and deciding a content type belong to what consumes and
  produces it.
- **Cross-origin handling, method override, and built-in middleware.** There is no cross-origin consumer, how methods
  are used is decided where actions are defined, and the middleware that exist arrive with what needs them.
- **Modules contributing middleware or routes**, which is a [collector](../GLOSSARY.md#collector) in the module system.

## Alternatives considered

The decisions this design rests on are recorded separately, with the alternatives each one rejected:

- bespoke messages rather than PSR-7 and PSR-15, in [ADR-0018](../adr/0018-http-messages-are-bespoke-not-psr-7.md)
- the panel being one server-rendered system, in
  [ADR-0001](../adr/0001-the-panel-is-a-single-system-not-a-panel-and-an-engine.md)
- sealing mutable registries into immutable catalogues, in
  [ADR-0003](../adr/0003-mutable-registries-are-sealed-into-immutable-catalogues.md)
- running as a worker, in [ADR-0009](../adr/0009-the-panel-runs-as-a-frankenphp-worker-in-a-single-binary.md)

**Using FastRoute's route collection as the route table.** We rejected this because it cannot be read back or run
backwards, and generation needs both.

**FastRoute's file-backed cached dispatcher.** We rejected this because the worker already holds the compiled form in
memory, so a file would add staleness for no gain.

**Composing every pipeline at boot.** We rejected this because which middleware apply to a route depends on the route
matched.

No other alternatives were weighed.

## Backwards compatibility

Nothing will break. The [engine](../GLOSSARY.md#engine) has no HTTP layer today.

## Open questions

- ~~**Where the panel context is defined.** Routing needs it, and it is currently defined inside the module system's
  collector work. It has to be promoted somewhere shared before routing can consume it.~~ Resolved by
  [RFC-0015](0015-module-system.md), which defines it in the [engine](../GLOSSARY.md#engine)'s shared values, alongside
  the client address.

## Changelog

- 2026-09-16: Where the panel context is defined is resolved by [RFC-0015](0015-module-system.md).
- 2026-10-05: Reworded in the readable house style, and the project's own vocabulary linked to the glossary. The
  design is unchanged.

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
