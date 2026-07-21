# quince

A thin HTTP server framework for Emela, distributed as a Pome.

`std.http`'s `HttpServer` capability (spec 0046) gives you `bind` / `accept` /
`respond` / `close` and expects you to write the accept loop yourself as a
self-tail-recursive `fn main`. That is a lot of boilerplate — the loop, the
`try`/`catch`, and hand-written `if req.url == ...` routing — and it assumes you
understand the effect system. quince wraps that into a `|>` builder (spec 0019)
so a server reads as a single pipeline.

```emela
import std.io
import std.http
import quince.response
import quince.router
import quince.server

fn home(req: Request) -> Response uses {} {
    response.ok("hello from quince\n")
}

fn hit(req: Request) -> Response uses { Io } {
    let logged = Io.print("hit /hit\n")
    response.ok("hit\n")
}

fn main() -> Unit uses { Io, Http, HttpServer } {
    router.new()
        |> router.get("/", home)
        |> router.get("/hit", hit)
        |> server.serve(8080)
}
```

## Handlers and their effects

A handler is an ordinary function `(Request) -> Response`. Write it at its
natural, minimal effect row: `home` above is pure (`uses {}`), `hit` uses
`{ Io }`, a proxy would use `{ Http }`. quince stores every route's handler at
the widest row it supports — `uses { Io, Http }` — and a narrower handler is
accepted there by **effect subsumption** (spec 0023: `{} ⊆ { Io, Http }`). You
never have to widen a handler by hand, and a pure handler stays pure for testing.

Because handlers are plain functions, you test them without a socket:

```emela
import brix.assert

@test
fn root_is_200() -> Unit uses {} {
    assert.eq(home(Request { method: Method::Get  url: "/"  headers: []  body: "" }).status, 200)
}
```

Run the suite with `emela test`.

## API

### `quince.response` — response builders (all `uses {}`)

| function | result |
|---|---|
| `ok(body)` | 200 with `body` |
| `text(code, body)` | `code` with `body` |
| `not_found()` | 404 with a default body |
| `status(res, code)` | `res` with the status replaced (`\|>` form) |
| `header(res, name, value)` | `res` with one header appended (`\|>` form) |

The `|>` forms take a `Response` first, so a handler can build one top-to-bottom:
`ok("x") |> status(201) |> header("content-type", "text/plain")`.

### `quince.router` — method + path routing

| function | result |
|---|---|
| `new()` | an empty router (fallback = 404) |
| `get(r, path, h)` / `post(r, path, h)` | append a route, return the router |
| `fallback(r, h)` | replace the fallback handler |
| `route(r, req)` | dispatch to the first matching route, else the fallback |
| `method_eq(a, b)` | compare two methods (no `impl Eq for Method`; see below) |

`new()` / `get` / `post` / `fallback` take the router first, so routes chain with
`|>`. Matching is method + exact path for now (query strings are compared as-is
until PR-B2 strips them).

### `quince.server` — the accept loop

| function | result |
|---|---|
| `serve(r, port)` | bind `port` and serve `r` forever |

`serve` takes the router first, so a chain ends with `... |> server.serve(8080)`.
It is the only module that touches an effect (`{ Io, Http, HttpServer }`).

## Capability audit

quince declares no dependencies at runtime (`std.http` / `std.io` are embedded
core, spec 0038; brix is a dev-dependency). A server built on quince has the
capability set `{ HttpServer, Io }` — the outbound-network capability `Http`
does **not** appear unless one of your handlers actually calls `Http.get`, so a
serve-only service reads as "listens, never dials out" straight from its
manifest (spec 0046 S1).

## Running the example

```
emela run examples/hello.emel --package .
curl localhost:8080/        # hello from quince
curl localhost:8080/hit     # hit   (also logs on the server)
curl localhost:8080/nope    # not found
```

## Limitations (this cut)

- **Fixed maximum row.** Handlers are stored at `uses { Io, Http }`. A handler
  needing some other effect (a user-defined `uses { Db }`) does not fit yet. That
  needs effect-row polymorphism (spec 0022) for a `serve<'e>` that carries the
  handler's own row — a later cut.
- **Sequential only.** One request at a time (spec 0046 S7); no concurrency.
- **Exact-path matching.** No path parameters (`/users/:id`) yet.
- `method_eq` is hand-written because the orphan rule (spec 0020) forbids
  `impl Eq for Method` here — `Method` is owned by `std.http`.

## Requirements

Needs an Emela toolchain with **function-value effect subsumption** (spec 0023):
storing a `uses {}` handler into a `uses { Io, Http }` field is what makes the
router usable. On a toolchain without it, only pure handlers can be registered.
