---
title: Testing
---

<!-- TODO(v2): New page for v2. Flesh out this stub before release. -->

Connect services are plain Go interfaces, so unit tests can call your handler
methods directly. To exercise the full RPC path — interceptors, codecs, and
error handling — without binding a port, use the in-process transport from
`connectrpc.com/connect/v2/connectinprocess`.

## In-process transport

<!-- TODO(v2): Expand on the example from the v2 announcement:

    func TestGreetService(t *testing.T) {
        server := connect.NewServer(validate.NewServerInterceptor())
        greetv1connect.RegisterGreetServiceHandler(server, &GreetServer{})

        client := connect.NewClient(connectinprocess.New(server))
        greetClient := greetv1connect.NewGreetServiceClient(client)

        res, err := greetClient.Greet(t.Context(), &greetv1.GreetRequest{Name: "Jane"})
        if err != nil {
            t.Fatalf("Greet: %v", err)
        }
        if got, want := res.GetGreeting(), "Hello, Jane!"; got != want {
            t.Errorf("Greeting = %q, want %q", got, want)
        }
    }

Cover:
- Why in-process beats httptest.Server/loopback listeners (speed, no flaky
  port management); when you still want a real HTTP round-trip.
- Message copying semantics: connectinprocess.WithCopyFunc and
  connectinprocess.ProtoCopy for full proto deep copies versus the default
  shallow copy.
- Streaming RPCs over the in-process transport.
- Production uses: delegating calls between services in the same process. -->

## Testing over HTTP

<!-- TODO(v2): Show the httptest.Server pattern with connecthttp.Mount and
connecthttp.NewTransport for tests that must cover the wire protocol (e.g.
custom codecs, compression, h2c). -->
