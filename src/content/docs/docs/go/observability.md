---
title: Observability
---

Connect stays close to `net/http`, which means any logging, tracing, or metrics that work with an `http.Handler` or `http.Client` will also work with Connect. In particular, the [otelhttp](https://pkg.go.dev/go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp) OpenTelemetry package integrates seamlessly with Connect servers and clients.

For more detailed, RPC-focused metrics, use the [otelconnect] package. [otelconnect] works with your [OpenTelemetry] metrics and tracing setup to capture information such as:

- `rpc.system`: Was this call gRPC, gRPC-Web, or Connect?
- `rpc.service` and `rpc.method`: What service and method was called?
- `responses_per_rpc`: How many messages were written to streaming responses?
- `error_code`/`status_code`: What specific gRPC or Connect error was returned?

OpenTelemetry can be quite complex, so this guide assumes that readers are familiar with:

- What [observability](https://opentelemetry.io/docs/concepts/observability-primer/) is.
- A basic understanding of [OpenTelemetry metrics and tracing](https://opentelemetry.io/docs/reference/specification/).
- How [TextMapPropagators](https://opentelemetry.io/docs/reference/specification/context/api-propagators/), [MeterProviders](https://opentelemetry.io/docs/reference/specification/metrics/sdk/), and [TraceProviders](https://opentelemetry.io/docs/concepts/signals/traces/) are initialized and used.

## Enabling OpenTelemetry for Connect

<!-- TODO(v2): Update for otelconnect v2 (connectrpc.com/otelconnect/v2).
Following the v2 interceptor split, the single otelconnect.NewInterceptor
becomes otelconnect.NewServerInterceptor and otelconnect.NewClientInterceptor,
passed directly to the constructors:

    otelServer, err := otelconnect.NewServerInterceptor()
    server := connect.NewServer(otelServer)
    greetv1connect.RegisterGreetServiceHandler(server, greeter)

    otelClient, err := otelconnect.NewClientInterceptor()
    client := connect.NewClient(
        connecthttp.NewTransport(http.DefaultClient, "http://localhost:8080"),
        otelClient,
    )
    greetClient := greetv1connect.NewGreetServiceClient(client)

Verify the exact constructor names and option set against the published
otelconnect v2 module before release — the v2 module also fixes the metrics
skew tracked in connect-go#665. Mention that connecthttp exposes per-call
send/receive stats (connecthttp.ServerInfoForContext(ctx).SendStats() /
ReceiveStats()) used by the new instrumentation. -->

Once you have OpenTelemetry set up in your application, enabling OpenTelemetry in a Connect project is as simple as adding the [otelconnect.NewInterceptor] option on Connect handler and client constructors. If you do not have OpenTelemetry in your application, you can refer to the [OpenTelemetry Go getting started guide](https://opentelemetry.io/docs/instrumentation/go/getting-started/).

```go mark={1,4-7,11,17}
import "connectrpc.com/otelconnect"

func main() {
	otelInterceptor, err := otelconnect.NewInterceptor()
	if err != nil {
		log.Fatal(err)
	}

	path, handler := greetv1connect.NewGreetServiceHandler(
		greeter,
		connect.WithInterceptors(otelInterceptor),
	)

	client := greetv1connect.NewGreetServiceClient(
		http.DefaultClient,
		"http://localhost:8080",
		connect.WithInterceptors(otelInterceptor),
	)
}
```

By default, this will use:

- TextMapPropagator from [otel.GetTextMapPropagator]
- MeterProvider from [otel.GetMeterProvider]
- TracerProvider from [otel.GetTracerProvider]

## Using custom MeterProvider, TraceProvider and TextMapPropagators

<!-- TODO(v2): Update the example below: the return type connect.Interceptor no
longer exists. The client constructor returns a connect.ClientInterceptor and
the server constructor a connect.ServerInterceptor; the options are unchanged
in shape. -->

When running multiple applications in a single binary, or if different sections of code should use different exporters, pass the correct exporters to [otelconnect.NewInterceptor] explicitly:

- [otelconnect.WithTracerProvider] to set the TracerProvider
- [otelconnect.WithMeterProvider] to set the MeterProvider
- [otelconnect.WithPropagator] to set the TextMapPropagator

```go
// newInterceptor instruments Connect clients and handlers using custom OpenTelemetry metrics, tracing, and propagation.
func newInterceptor(
	tracerProvider trace.TracerProvider,
	metricProvider metric.MeterProvider,
	textMapPropagator propagation.TextMapPropagator
) (connect.Interceptor, error) {
	return otelconnect.NewInterceptor(
		otelconnect.WithTracerProvider(tracerProvider),
		otelconnect.WithMeterProvider(metricProvider),
		otelconnect.WithPropagator(textMapPropagator),
	)
}
```

## Configuration for internal microservices

By default, [otelconnect]-instrumented servers are conservative and behave as though they're internet-facing. They don't trust any tracing information sent by the client, and will create new trace spans for each request. The new spans are linked to the remote span for reference (using OpenTelemetry's [trace.Link]), but tracing UIs will display the request as a new top-level transaction.

If your server is deployed as an internal microservice, configure [otelconnect] to trust the client's tracing information using [otelconnect.WithTrustRemote]. With this option, servers will create child spans for each request.

## Reducing metrics and tracing cardinality

By default, the [OpenTelemetry RPC conventions](https://opentelemetry.io/docs/specs/semconv/rpc/rpc-spans/) produce high-cardinality server-side metric and tracing output. In particular, servers tag all metrics and trace data with the server's IP address and the remote port number. To drop these attributes, use [otelconnect.WithoutServerPeerAttributes]. For more customizable attribute filtering, use [otelconnect.WithFilter].

[otelconnect]: https://pkg.go.dev/connectrpc.com/otelconnect
[OpenTelemetry]: https://opentelemetry.io/
[trace.Link]: https://pkg.go.dev/go.opentelemetry.io/otel/trace#Link
[otel.GetMeterProvider]: https://pkg.go.dev/go.opentelemetry.io/otel#GetMeterProvider
[otel.GetTracerProvider]: https://pkg.go.dev/go.opentelemetry.io/otel#GetTracerProvider
[otel.GetTextMapPropagator]: https://pkg.go.dev/go.opentelemetry.io/otel#GetTextMapPropagator
[otelconnect.WithTracerProvider]: https://pkg.go.dev/connectrpc.com/otelconnect#WithTracerProvider
[otelconnect.WithMeterProvider]: https://pkg.go.dev/connectrpc.com/otelconnect#WithMeterProvider
[otelconnect.WithPropagator]: https://pkg.go.dev/connectrpc.com/otelconnect#WithPropagator
[otelconnect.NewInterceptor]: https://pkg.go.dev/connectrpc.com/otelconnect#NewInterceptor
[otelconnect.WithTrustRemote]: https://pkg.go.dev/connectrpc.com/otelconnect#WithTrustRemote
[otelconnect.WithFilter]: https://pkg.go.dev/connectrpc.com/otelconnect#WithFilter
[otelconnect.WithoutServerPeerAttributes]: https://pkg.go.dev/connectrpc.com/otelconnect#WithoutServerPeerAttributes
