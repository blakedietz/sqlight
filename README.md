# sqlight

[![Package Version](https://img.shields.io/hexpm/v/sqlight)](https://hex.pm/packages/sqlight)
[![Hex Docs](https://img.shields.io/badge/hex-docs-ffaff3)](https://hexdocs.pm/sqlight/)

Use [SQLite](https://www.sqlite.org/index.html) from Gleam!

Works on Erlang or JavaScript running on Deno.

> **Fork note — `sqlight.load_extension`**
>
> This fork adds `sqlight.load_extension(path:, entrypoint:, on:)` for loading
> SQLite run-time extensions, and depends on a matching
> [esqlite fork](https://github.com/blakedietz/esqlite/tree/load-extension)
> (the stock Hex `esqlite` does not export `load_extension/3`).
>
> ## Using this fork
>
> Point both dependencies at the forks in your `gleam.toml`:
>
> ```toml
> [dependencies]
> sqlight = { git = "https://github.com/blakedietz/sqlight.git", ref = "load-extension" }
> ```
>
> `esqlite` is a rebar3 package with a C NIF. Gleam compiles that NIF
> automatically **only for Hex dependencies** — when esqlite is pulled in as a
> **git** dependency (as it is here), Gleam does not run rebar3, so you must
> build the NIF yourself after fetching deps:
>
> ```sh
> gleam deps download
> (cd build/packages/esqlite && make compile)   # builds priv/esqlite3_nif.so
> gleam build                                    # or: gleam test --target erlang
> ```
>
> Re-run the `make compile` step whenever `build/packages/` is regenerated
> (e.g. after changing dependencies or a clean checkout). A C toolchain and
> `rebar3` must be available. See `.github/workflows/test.yml` for a working
> CI example. The JavaScript target does not support extension loading and
> `load_extension` returns an error there.

```sh
gleam add sqlight
```

```gleam
import gleam/dynamic/decode
import sqlight

pub fn main() {
  use conn <- sqlight.with_connection(":memory:")

  let sql = "
  create table cats (name text, age int);

  insert into cats (name, age) values 
  ('Nubi', 4),
  ('Biffy', 10),
  ('Ginny', 6);
  "
  let assert Ok(Nil) = sqlight.exec(sql, conn)

  let cat_decoder = {
    use name <- decode.field(0, decode.string)
    use age <- decode.field(1, decode.int)
    decode.success(#(name, age))
  }

  let sql = "
  select name, age from cats
  where age < ?
  "
  let assert Ok([#("Nubi", 4), #("Ginny", 6)]) =
    sqlight.query(sql, on: conn, with: [sqlight.int(7)], expecting: cat_decoder)
}
```

### Loading a SQLite extension

`load_extension` loads a SQLite run-time extension (a shared library) onto a
connection. Extension loading is enabled only for the duration of the call.
Pass an empty `entrypoint` to let SQLite derive the entry point from the
filename, or name it explicitly. This is supported on the Erlang target only;
on JavaScript it returns an error.

```gleam
import gleam/dynamic/decode
import sqlight

pub fn main() {
  use conn <- sqlight.with_connection("my.db")

  // Load an extension. The path is to the shared library, without the
  // platform-specific suffix (.so/.dylib/.dll), which SQLite appends.
  let assert Ok(Nil) =
    sqlight.load_extension(path: "./uuid", entrypoint: "", on: conn)

  // The functions provided by the extension are now available in SQL.
  let uuid_decoder = decode.at([0], decode.string)
  let assert Ok([_uuid]) =
    sqlight.query(
      "select uuid()",
      on: conn,
      with: [],
      expecting: uuid_decoder,
    )
}
```

Documentation can be found at <https://hexdocs.pm/sqlight>.

## Why SQLite?

SQLite is a implementation of SQL as a library. This means that you don't run a
separate SQL server that your program communicates with, but you embed the SQL
implementation directly in your program. SQLite stores its data in a single
file. The file format is portable between different machine architectures. It
supports atomic transactions and it is possible to access the file by multiple
processes and different programs.

You can also use in-memory databases with SQLite, which may be useful for testing.

## Implementation

When running on Erlang it is a library wrapper around the excellent Erlang library
[esqlite](https://hex.pm/packages/esqlite), which in turn is a wrapper around
the SQLite C library. It is implemented as a NIF, which means that the SQLite
database engine is linked to the erlang virtual machine.

When running on Deno it is a wrapper around the excellent
[x/sqlite](https://deno.land/x/sqlite@v3.7.0) library, which in turn is a
wrapper around the SQLite C library compiled to WASM.

## On using Bool with SQLite

SQLite does not have a native boolean type. Instead, it uses ints, where 0 is
False and 1 is True. Because of this the Gleam stdlib decoder for bools will not
work, instead the `sqlight.decode_bool` function should be used as it supports
both ints and bools.
