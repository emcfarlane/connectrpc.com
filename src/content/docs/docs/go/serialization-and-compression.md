---
title: Serialization & compression
---

Connect can work with any schema definition language, but it comes with
built-in support for Protocol Buffers. Because the Protocol Buffer
specification includes [mappings to and from JSON][protojson], any Connect API
defined with a Protobuf schema also supports JSON. This is especially
convenient for web browsers and ad-hoc debugging with cURL.

<!-- TODO(v2): Rewrite for the v2 codec layout:
- The built-in Protobuf binary and JSON codecs now live in
  connectrpc.com/connect/v2/connectproto (connectproto.NewBinaryCodec(),
  connectproto.NewJSONCodec(), both accepting WithTypeResolver). connecthttp
  installs them by default.
- WithProtoJSON() is replaced by selecting the send codec on the transport:
  connecthttp.WithSendCodec(connect.CodecNameJSON).
- WithCodec is replaced by connecthttp.WithCodecs(codecs...), which registers
  codecs by their Name() on transports and servers.
- The connect.Codec interface changed shape: MarshalAppend(ctx, dst, msg) and
  Unmarshal(ctx, src, msg) — methods take a context and use append-style
  buffers. StableCodec (MarshalAppendStable, IsBinary) is required for GET
  support. Custom codec docs need a full update. -->

Connect handlers automatically accept JSON-encoded requests &mdash; there's no
special configuration required. Connect clients default to using binary
Protobuf. To configure your client to use JSON instead, use the
`WithProtoJSON()` option during client construction.

## Replacing standard Protobuf

By default, Connect uses `google.golang.org/protobuf` to serialize and
deserialize Protobuf messages. To use a different Protobuf runtime, implement
the `Codec` interface using the `"proto"` name. Then pass your implementation
to your handlers and clients using the `WithCodec` option. Connect will use
your custom codec to marshal and unmarshal a variety of unexported,
protocol-specific messages, so take care to fall back to the standard Protobuf
runtime if necessary.

## Custom serialization

To support a completely different serialization mechanism, you'll first need to
implement `Codec`. If your new serialization mechanism uses a schema, you'll
also need to write a binary to generate RPC code from the schema. Typically,
this binary is a plugin for the appropriate compiler (for example,
`thriftrw-go` for Thrift). This isn't as complex as it may sound! Because it
uses Go type parameters, Connect doesn't require much generated code.

## Compression

Connect clients and handlers support compression. Usually, compression is
helpful &mdash; the small increase in CPU usage is more than offset by the
reduction in network I/O.

In particular, Connect encourages _asymmetric_ compression: clients can send
uncompressed requests while asking for compressed responses. Because responses
are usually larger than requests, this approach compresses most of the data on
the network without requiring the client to make any assumptions about the
server.

<!-- TODO(v2): Update for the v2 compression layout:
- gzip now lives in connectrpc.com/connect/v2/connectgzip
  (connectgzip.New(), with a WithLevel option). connecthttp installs it by
  default.
- WithSendGzip is replaced by
  connecthttp.WithSendCompressor(connect.CompressionNameGzip).
- WithCompressMinBytes moved verbatim to connecthttp.WithCompressMinBytes. -->

By default, Connect handlers support gzip compression using the standard
library's `compress/gzip` at the default compression level. Connect clients
default to sending uncompressed requests and asking for gzipped responses. If
you know that the server supports gzip, you can also compress requests by using
the `WithSendGzip` option during client construction.

Like most compression schemes, gzip _increases_ the size of very small
messages. By default, Connect handlers (and clients using `WithSendGzip`)
compress messages without considering their size. To only compress messages
larger than some threshold, use `WithCompressMinBytes` during handler and
client construction. In most cases, this improves overall performance.

Finally, it's worth noting that clients using the Connect protocol for unary
RPCs ask for compressed responses using the `Accept-Encoding` HTTP header. This
matches standard HTTP semantics, so browsers can easily make efficient Connect
RPCs: they automatically ask for compressed responses, and the network
inspector tab automatically decompresses the data if necessary. Connect's
`Accept-Encoding` support also works well with cURL's `--compressed` flag.

## Custom compression

<!-- TODO(v2): Rewrite for the v2 compressor API:
- The separate Compressor/Decompressor interfaces are replaced by a single
  connect.Compressor interface: Name(), CompressAppend(ctx, dst, src), and
  DecompressAppend(ctx, dst, src).
- WithCompression and WithAcceptCompression are replaced by
  connecthttp.WithCompressors(compressors...), used for both transports and
  servers; WithSendCompression becomes connecthttp.WithSendCompressor(name).
- connect-go now ships a Zstandard implementation in
  connectrpc.com/connect/v2/connectzstd — mention it as the worked example
  instead of (or alongside) hand-rolling. -->

In Go, Connect comes with gzip support because it's widely used and included in
the standard library. To support newer compression algorithms, like Brotli or
Zstandard, first implement the `Compressor` and `Decompressor` interfaces.
Configure your handlers with `WithCompression`, and configure your clients with
`WithAcceptCompression`. Where appropriate, take care to use the [IANA name for
your compression algorithm][iana-compression] (for example, `br` for Brotli and
`zstd` for Zstandard). To have your client also send compressed requests, use
`WithSendCompression`.

[protojson]: https://developers.google.com/protocol-buffers/docs/proto3#json
[iana-compression]: https://www.iana.org/assignments/http-parameters/http-parameters.xml#content-coding
