---
title: Errors
---

Similar to the familiar "404 Not Found" and "500 Internal Server Error" status
codes you may have seen in HTTP, Connect uses a set of [16 error
codes](/docs/protocol/#error-codes). The Go APIs for creating and inspecting
errors work identically for all three supported protocols: gRPC, gRPC-Web, and
the Connect protocol.

## Working with errors

<!-- TODO(v2): Rewrite this section for the new error semantics. This is a real
behavior change, not just a rename:
- connect.NewError now takes a string message: NewError(code Code, message
  string). connect.Errorf(code, format, args...) is the printf variant.
- Only the explicit message is serialized to the wire. A bare error returned
  from a handler becomes CodeUnknown with NO wire message — v1's behavior of
  sending err.Error() verbatim is gone. Call out the privacy flip.
- Attach an underlying error with (*Error).WithCause(err). The cause is
  available to errors.Is/errors.As locally but is never serialized.
- Errors received from a client call are marked remote ((*Error).IsRemote()).
  Returning a remote error from a handler is rewritten to a bare CodeInternal.
  Handlers must translate downstream errors into explicit local errors. -->

At their simplest, `connect-go` errors attach an [error
code](/docs/protocol/#error-codes) to a standard Go error. The error code and
the underlying error's `Error()` string are sent over the network to the
client, which may handle different codes with different retry or fallback
logic. If you're familiar with gRPC status codes, Connect's error codes use the
same names and have the same semantics.

Connect handlers attach status codes to errors using the `NewError` function.
Handlers should return coded errors; if they don't, Connect will use the
`deadline_exceeded` code for `context.DeadlineExceeded`, `canceled` for
`context.Canceled`, and `unknown` for all other errors. For example:

```go
func (s *GreetServer) Greet(
	ctx context.Context,
	req *greetv1.GreetRequest,
) (*greetv1.GreetResponse, error) {
	if err := ctx.Err(); err != nil {
		return nil, err // automatically coded correctly
	}
	if err := validateGreetRequest(req); err != nil {
		return nil, connect.NewError(connect.CodeInvalidArgument, err)
	}
	greeting, err := doGreetWork(ctx, req)
	if err != nil {
		return nil, connect.NewError(connect.CodeUnknown, err)
	}
	return &greetv1.GreetResponse{
		Greeting: greeting,
	}, nil
}
```

Regardless of the protocol in use, Connect clients automatically unmarshal
response data into a standard Go error. Use Connect's `CodeOf` function and the
standard library's `errors.As` to inspect errors:

<!-- TODO(v2): Update the client example below: construct the client with
connect.NewClient(connecthttp.NewTransport(http.DefaultClient,
"http://localhost:8080")) and pass it to greetv1connect.NewGreetServiceClient.
Mention (*Error).IsRemote() for distinguishing the peer's verdict from local
errors (replaces v1 IsWireError). -->

```go
client := greetv1connect.NewGreetServiceClient(
	http.DefaultClient,
	"http://localhost:8080",
)
_, err := client.Greet(
	context.Background(),
	&greetv1.GreetRequest{
		Name: "Jane",
	},
)
if err != nil {
	fmt.Println(connect.CodeOf(err))
	if connectErr := new(connect.Error); errors.As(err, &connectErr) {
		fmt.Println(connectErr.Message())
		fmt.Println(connectErr.Details())
	}
}
```

These APIs work for all three supported protocols, even if the server isn't
built with Connect.

## Error Details

<!-- TODO(v2): Rewrite this section for the v2 details API:
- The ErrorDetail wrapper type, NewErrorDetail, and AddDetail are gone.
- Servers attach details with (*Error).WithDetail(detail any), which returns a
  cloned error: err = err.WithDetail(retryInfo). For connecthttp protocols, a
  detail may be a proto.Message (wrapped in Any at encode time) or an
  *anypb.Any.
- (*Error).Details() returns []any. The concrete detail types are
  transport-defined. For connecthttp's Connect/gRPC/gRPC-Web protocols,
  received details are *anypb.Any values, so the client example must unmarshal
  with anypb.UnmarshalNew (or check TypeUrl) instead of detail.Value(). -->

Like `grpc-go`, `connect-go` allows servers to enrich errors with more than
just a code and a string. Since Connect focuses on schema-first APIs, this
additional data &mdash; called error details &mdash; is a slice of Protobuf
messages wrapped in the `ErrorDetail` type. Details are commonly used to send
backoff parameters for transient failures, localized error messages, or other
structured data. The `google.golang.org/genproto/googleapis/rpc/errdetails`
package contains a variety of Protobuf messages often used as error details.
Regardless of the RPC protocol in use, servers can add details to any `*Error`:

```go
package example

import (
	"errors"

	"connectrpc.com/connect"
	"google.golang.org/genproto/googleapis/rpc/errdetails"
	"google.golang.org/protobuf/types/known/durationpb"
)

func newTransientError() error {
	err := connect.NewError(
		connect.CodeUnavailable,
		errors.New("overloaded: back off and retry"),
	)
	retryInfo := &errdetails.RetryInfo{
		RetryDelay: durationpb.New(10 * time.Second),
	}
	if detail, detailErr := connect.NewErrorDetail(retryInfo); detailErr == nil {
		err.AddDetail(detail)
	}
	return err
}
```

Clients receive error details as a slice of `*ErrorDetail`, which they must
inspect to find any details of interest:

```go
package example

import (
	"errors"

	"connectrpc.com/connect"
	"google.golang.org/genproto/googleapis/rpc/errdetails"
	"google.golang.org/protobuf/types/known/durationpb"
)

func extractRetryInfo(err error) (*errdetails.RetryInfo, bool) {
	var connectErr *connect.Error
	if !errors.As(err, &connectErr) {
		return nil, false
	}
	for _, detail := range connectErr.Details() {
		msg, valueErr := detail.Value()
		if valueErr != nil {
			// Usually, errors here mean that we don't have the schema for this
			// Protobuf message.
			continue
		}
		if retryInfo, ok := msg.(*errdetails.RetryInfo); ok {
			return retryInfo, true
		}
	}
	return nil, false
}
```

Error details work best if they're limited to a small set of stable types used
by all APIs within your organization. Used sparingly, they're safer, more
extensible, and more efficient than ad-hoc HTTP header microformats.

Again, the handler and client APIs for working with error details work the same
for gRPC, gRPC-Web, and the Connect protocol.

## HTTP representation

While the Go APIs for working with errors are protocol-agnostic, each protocol
produces a differently-shaped HTTP response. You can consult the [gRPC HTTP/2
protocol][grpc-protocol], the [gRPC-Web protocol][grpcweb-protocol], and the
[Connect protocol](/docs/protocol/) for details, but it's helpful to understand
the broad strokes of each protocol's approach.

gRPC responses nearly always have an HTTP status of 200 OK, even when the
server returns an error. The error's code and message are sent as separate HTTP
trailers. The code, message, and error details are also serialized with
Protobuf, base64-encoded, and put into a third HTTP trailer. This approach is
the same for unary and streaming RPCs, and because trailers are part of the
HTTP standard, it's theoretically possible for any HTTP library to work with
the error code and message. Unfortunately, trailer support is spotty &mdash;
even web browsers don't support them.

gRPC-Web responses use nearly the same approach, but encode all the trailers
into the last portion of the response body. This works even when clients don't
support HTTP trailers, but still leaves failed responses with an HTTP 200
status code.

For unary (request-response) RPCs, the Connect protocol puts error codes,
messages, and details in the response body as human-readable JSON. The
response's HTTP status code is inferred from the Connect code, and is always in
the 4xx or 5xx range if the RPC fails. The response body for a unary Connect
error might look like this:

```json
{
  "code": "invalid_argument",
  "message": "data cannot be empty"
}
```

For streaming RPCs, the Connect protocol handles errors similarly to gRPC-Web.
This two-pronged approach sacrifices consistency between streaming and unary
RPCs, but it stays as close as possible to standard HTTP semantics and works
with the broadest array of HTTP libraries and tools.

[grpc-protocol]: https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-HTTP2.md
[grpcweb-protocol]: https://github.com/grpc/grpc/blob/master/doc/PROTOCOL-WEB.md
