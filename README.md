# mah-todo-demo

A todo list with a REST API, an OpenAPI document and Swagger UI, written
entirely in [**Mah**](https://github.com/sina-salahshour/mahlang), a small
dynamically typed language with closures, structs, enums, traits,
decorators and run-time reflection.

There is no web framework underneath: the HTTP server is ~300 lines of Mah
on top of `std:socket`, the routes are plain functions with decorators, and
the OpenAPI document is written at run time from those functions'
declarations with `std:reflect`.

> Made with [Mah](https://github.com/sina-salahshour/mahlang) ☾ ·
> [mahlang.dev](https://mahlang.dev)

## Run it

You need the `mah` command (see [installing Mah](https://github.com/sina-salahshour/mahlang#readme)).

```sh
mah run                       # http://127.0.0.1:4000 (or the next free port)
mah run -- --port 9000        # pick a port; HOST and PORT in the environment work too
mah run -- --host 0.0.0.0     # listen on every interface
```

Then open:

| URL | what |
|---|---|
| `/` | the todo list (light and dark themes, filters, double-click to rename, "clear done") |
| `/docs` | Swagger UI over the API |
| `/openapi.json` | the OpenAPI 3.0 document |
| `/api` | what the service is, and every route |
| `/health` | `{"status": "ok", "todos": N}` |

The todos live in a variable for as long as the program runs: nothing is
written to disk, so every run starts empty.

## The API

| method | path | does |
|---|---|---|
| `GET` | `/todos?done=true\|false` | list todos (all, or only finished / only open) |
| `POST` | `/todos` | add one: `{"title": "buy milk", "done": false}` → `201` + `Location` |
| `GET` | `/todos/{id}` | one todo, or `404` |
| `PUT` | `/todos/{id}` | replace the title and the done flag |
| `PATCH` | `/todos/{id}` | change only the fields given |
| `POST` | `/todos/{id}/toggle` | flip done |
| `DELETE` | `/todos/{id}` | delete one → `204` |
| `DELETE` | `/todos/done` | delete every finished todo → `{"removed": N}` |

Errors are always `{"error": "<code>", "message": "<words>"}` with a fitting
status (`400`, `404`, `405`, `422`, `500`).

```sh
curl -s localhost:4000/todos -H 'content-type: application/json' -d '{"title": "buy milk"}'
curl -s 'localhost:4000/todos?done=false'
curl -s -X PATCH localhost:4000/todos/1 -d '{"done": true}'
```

## How it works

A route is one function. Its decorator says the method and path, its
parameters say where each value comes from, and its `##` doc comment and
types say everything the documentation needs:

```mah
## Get one todo.
@get("/todos/{id}", errors: [404], tags: ["todos"])
fn get_todo(
    db: store.Store,
    ## The todo's id.
    @path id: Number
) -> store.Todo {
    let todo = db.get(id)
    if todo == none { throw missing(id) }
    todo.unwrap()
}
```

- **`src/rest.mh`**: the route decorators (`@get`, `@post`, ...) are
  `std:reflect` *hooks*: as the program starts, each one is handed the
  function it decorates and registers it. `mount` then reads every
  function's signature with `reflect.signature` and, per request, fills
  `@path`/`@query` parameters converted to their declared type, a `@body`
  read with `json.parse_as` into its declared struct, and any other
  parameter with the value of that type it was given (the store). The
  result becomes the response: a struct is JSON, `Created<T>` is a `201`
  with a `Location`, `-> None` is a `204`, and a thrown `ApiError` is its
  status.
- **`src/openapi.mh`**: walks the same signatures, plus `reflect.schema` of
  every struct they name, to write the OpenAPI document. Doc comments
  become summaries and descriptions, field docs and `@example(...)`
  decorators become schema descriptions and examples. Nothing is written
  by hand per route, so the docs can't drift from the code.
- **`src/http.mh`**: reads requests and writes responses over
  `std:socket`, decodes query strings and path parameters with `std:url`,
  and routes `METHOD /path/{param}` patterns.
- **`src/app.mh`**: the command line, the listening socket (moving up to
  the next port when one is taken) and the accept loop, one task per
  connection.
- **`src/store.mh`**: the todos, in memory.
- **`src/web.mh`**: the two pages, as strings, so `mah run` is all it takes.

## Tests

```sh
mah test              # every *.test.mh, including end-to-end tests that
                      # drive the real server with the std:http client
mah test --vm python  # the same on the reference VM
mah check             # the static type checker
mah format --check    # the standard layout
```

## Build and deploy

```sh
mah build --target release      # build/todo.mahc, a standalone executable
cp build/todo.mahc out/todo.mahc
docker build -t mah-todo .      # copies out/todo.mahc into debian:trixie-slim
docker run -p 8080:8080 -e PORT=8080 mah-todo
```

`out/todo.mahc` is committed so the Dockerfiles work without Mah
installed; rebuild it when the code changes.
