---
title: Migrating to v2
---

<!-- TODO(v2): Flesh out this stub before release. Model it on
docs/web/migrating-to-v2.mdx: a short summary of the headline changes plus a
pointer to the MIGRATING.md guide in the connect-go repository. Draft content
below; verify links once v2.0 and MIGRATING.md are published. -->

Version 2 of connect-go is available. The key changes are:

- **Simple signatures are now the default.** The `connect.Request` and
  `connect.Response` wrappers are gone; unary handlers and clients use plain
  Protobuf messages, and the generator's `simple` flag has been removed.
- **Transports are pluggable.** Generated code no longer depends on
  `net/http`. Servers register with `connect.NewServer` and are mounted with
  `connecthttp.Mount`; clients wrap a `connect.Transport` created by
  `connecthttp.NewTransport`. An in-process transport,
  `connectinprocess`, makes testing fast — no listeners or loopback HTTP.
- **Interceptors are unified.** The three-method `Interceptor` interface is
  replaced by `connect.ClientInterceptor` and `connect.ServerInterceptor`
  function types that work for unary and streaming RPCs alike.
- **Error semantics are explicit.** `connect.NewError` takes a message string,
  bare errors are no longer serialized to the wire, and errors received from
  clients are marked remote.
- **Metadata moves to context.** Headers and trailers are reached through a
  `CallInfo` via `connect.NewClientContext` and
  `connect.ServerInfoForContext`.

<!-- TODO(v2): Add a section on the migration tool:

    go install connectrpc.com/connect/v2/cmd/connect-migrate-go@latest

It analyzes dependencies, code generation config, and Go sources, prompting
with diffs for the mechanical changes. -->

<!-- TODO(v2): Link the authoritative migration guide once published, e.g.
https://github.com/connectrpc/connect-go/blob/main/MIGRATING.md, and the v2
announcement blog post. Note that v1 remains supported on a maintenance
branch. -->
