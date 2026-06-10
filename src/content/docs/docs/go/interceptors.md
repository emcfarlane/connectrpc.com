---
title: Interceptors
---

Interceptors are similar to the middleware or decorators you may be familiar
with from other frameworks: they're the primary way of extending Connect and are
often used to add logging, metrics, tracing, retries, and other
functionality.
If you followed the [getting started](/docs/go/getting-started/) guide, you've already seen an interceptor in action:
the [validate-go](https://github.com/connectrpc/validate-go/) interceptor powers the Protovalidate integration that made sure every `GreetRequest` contained a valid name.

<!-- TODO(v2): Rewrite this page for the unified interceptor model. There is no
unary/streaming split anymore: every RPC — including unary — is modeled as a
stream, and interceptors come in client and server flavors:

    type ClientFunc func(ctx context.Context, spec Spec, stream ClientStream) error
    type ServerFunc func(ctx context.Context, spec Spec, stream ServerStream) error
    type ClientInterceptor func(next ClientFunc) ClientFunc
    type ServerInterceptor func(next ServerFunc) ServerFunc

UnaryFunc, AnyRequest/AnyResponse, UnaryInterceptorFunc, and the Interceptor
interface are gone, as is Spec().IsClient — the client/server distinction is
now in the type system. Explain the unary-as-stream model: a unary RPC is a
stream that sends and receives exactly one message, and interceptors that need
to inspect messages wrap the stream before calling next. Headers are reached
through CallInfo (connect.ClientInfoForContext / connect.ServerInfoForContext).
Use the slog logging interceptor from the v2 announcement as the example. -->

On this page you'll learn how to build unary interceptors &mdash; more complex use
cases are covered in the [streaming documentation](/docs/go/streaming/).

Take care when writing interceptors! They're powerful, but overly complex
interceptors can make debugging difficult.

## Interceptors are functions

Unary interceptors are built on two interfaces: `AnyRequest` and `AnyResponse`
and provide access to the request and response data only as an `any`. With these
interfaces, we can model all unary RPCs as:

```go
type UnaryFunc func(context.Context, AnyRequest) (AnyResponse, error)
```

An interceptor wraps an RPC with some additional logic, so it's transforming
one `UnaryFunc` into another:

```go
type UnaryInterceptorFunc func(UnaryFunc) UnaryFunc
```

Most unary interceptors are best implemented as a `UnaryInterceptorFunc`.

## An example

That's a little abstract, so let's consider an example: we'd like to apply a
simple header-based authentication scheme to our RPCs. We could add this logic
to each method on our server, but it's less error-prone to write an interceptor
instead.

```go
package example

import (
	"context"
	"errors"

	"connectrpc.com/connect"
)

const tokenHeader = "Acme-Token"

func NewAuthInterceptor() connect.UnaryInterceptorFunc {
	return func(next connect.UnaryFunc) connect.UnaryFunc {
		return func(
			ctx context.Context,
			req connect.AnyRequest,
		) (connect.AnyResponse, error) {
			if req.Spec().IsClient {
				// Send a token with client requests.
				req.Header().Set(tokenHeader, "sample")
			} else if req.Header().Get(tokenHeader) == "" {
				// Check token in handlers.
				return nil, connect.NewError(
					connect.CodeUnauthenticated,
					errors.New("no token provided"),
				)
			}
			return next(ctx, req)
		}
	}
}
```

To apply our new interceptor to handlers or clients, we can use
`WithInterceptors`:

<!-- TODO(v2): connect.WithInterceptors is gone. Interceptors are passed
directly to the constructors:

    // For handlers:
    server := connect.NewServer(
        NewAuthInterceptor(),
        validate.NewServerInterceptor(),
    )
    greetv1connect.RegisterGreetServiceHandler(server, &GreetServer{})

    // For clients:
    client := connect.NewClient(
        connecthttp.NewTransport(http.DefaultClient, "http://localhost:8080"),
        NewAuthInterceptor(),
    )

Interceptors fire in argument order; the first wraps the outermost call. -->

```go
// For handlers:
interceptors := connect.WithInterceptors(
	NewAuthInterceptor(),
	validate.NewInterceptor(),
)
mux := http.NewServeMux()
mux.Handle(greetv1connect.NewGreetServiceHandler(
	&GreetServer{},
	interceptors,
))
```

```go
// For clients:
client := greetv1connect.NewGreetServiceClient(
	http.DefaultClient,
	"http://localhost:8080",
	connect.WithInterceptors(NewAuthInterceptor()),
)
```
