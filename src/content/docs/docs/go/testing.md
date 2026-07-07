---
title: Testing
---

Connect services are plain Go interfaces, so unit tests can call your handler
methods directly. To exercise the full RPC path &mdash; interceptors, codecs,
and error handling &mdash; without binding a port, use the in-process
transport from `connectrpc.com/connect/v2/connectinprocess`.

## In-process transport

The in-process transport dispatches RPCs directly to a `connect.Server`
&mdash; no listeners, no loopback HTTP, and no flaky port management. It runs
the same interceptor and error paths as a network transport:

```go
func TestGreetService(t *testing.T) {
	server := connect.NewServer()
	greetv1connect.RegisterGreetServiceHandler(server, &GreetServer{})

	client := greetv1connect.NewGreetServiceClient(
		connect.NewClient(connectinprocess.New(server)),
	)

	res, err := client.Greet(t.Context(), &greetv1.GreetRequest{Name: "Jane"})
	if err != nil {
		t.Fatalf("Greet: %v", err)
	}
	if got, want := res.GetGreeting(), "Hello, Jane!"; got != want {
		t.Errorf("Greeting = %q, want %q", got, want)
	}
}
```

Streaming RPCs work the same way, using the generated stream types.

By default, messages are deep-copied between the client and the server with
`connectinprocess.ProtoCopy`, so tests behave like a network transport even
when a handler mutates its request. Use `connectinprocess.WithCopyFunc` to
override the copy strategy.

The in-process transport isn't limited to tests. It's also useful in
production for delegating calls between services running in the same process.

## Testing over HTTP

Some tests must cover the wire protocol &mdash; custom codecs, compression,
or h2c configuration. For those, mount the server on an `httptest.Server` and
point a `connecthttp` transport at it:

```go
func TestGreetServiceHTTP(t *testing.T) {
	server := connect.NewServer()
	greetv1connect.RegisterGreetServiceHandler(server, &GreetServer{})
	mux := http.NewServeMux()
	connecthttp.Mount(mux, server)
	ts := httptest.NewServer(mux)
	defer ts.Close()

	client := greetv1connect.NewGreetServiceClient(
		connect.NewClient(connecthttp.NewTransport(ts.Client(), ts.URL)),
	)

	res, err := client.Greet(t.Context(), &greetv1.GreetRequest{Name: "Jane"})
	if err != nil {
		t.Fatalf("Greet: %v", err)
	}
	if got, want := res.GetGreeting(), "Hello, Jane!"; got != want {
		t.Errorf("Greeting = %q, want %q", got, want)
	}
}
```
