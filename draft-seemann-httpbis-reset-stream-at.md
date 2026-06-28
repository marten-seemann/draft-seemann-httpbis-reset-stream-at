---
title: "Using QUIC Stream Resets with Partial Delivery in HTTP/3"
abbrev: "Reliable QUIC Stream Resets in HTTP/3"
category: info

docname: draft-seemann-httpbis-reset-stream-at-latest
ipr: trust200902
submissiontype: IETF
consensus: true
v: 3
area: "Web and Internet Transport"
workgroup: "HTTP"


author:
 -
    fullname: Marten Seemann
    email: martenseemann@gmail.com

normative:
  RESET-STREAM-AT: I-D.ietf-quic-reliable-stream-reset
  HTTP3: RFC9114

informative:
  WEBTRANS-HTTP3: I-D.ietf-webtrans-http3

--- abstract

QUIC stream resets do not guarantee delivery of stream data sent before the
reset.  This is a problem for HTTP/3 when the appropriate protocol outcome is an
abrupt stream termination, but the sender also has a bounded prefix that is
useful to deliver.  In HTTP/3, that prefix is often an HTTP response header
block, optionally followed by a short diagnostic response body, or a partial
response received from an upstream host.

WebTransport over HTTP/3 uses RESET_STREAM_AT to ensure that stream prefix
information is delivered when a WebTransport stream is reset.  That use case
motivated the QUIC extension, and many HTTP/3 implementations already support
it for that reason.  This document records additional HTTP/3 use cases for the
same mechanism.

This document describes HTTP/3 use cases for the RESET_STREAM_AT.  These include
CONNECT tunnels, intermediary forwarding failures after a response has been
committed, malformed decoded HTTP messages, and requests rejected before
application processing.


--- middle

# Introduction

HTTP/3 uses QUIC streams to carry requests and responses {{HTTP3}}. Several
HTTP/3 errors are signaled by abruptly terminating a stream with an HTTP/3 error
code.  However, a QUIC stream reset does not ensure delivery of stream data sent
before the reset.

This creates a gap when an endpoint has useful information to send, but the
correct HTTP/3 signal is still a stream error.  A proxy can have received
response bytes from an upstream connection before learning that the upstream
connection failed.  A server can also decode a request field section, determine
that the HTTP message is malformed, and generate a short diagnostic response
before terminating the stream with H3_MESSAGE_ERROR.

RESET_STREAM_AT addresses this by adding a Reliable Size to the reset. The
stream still terminates abruptly, but stream data up to the Reliable Size is
delivered reliably.

WebTransport over HTTP/3 uses RESET_STREAM_AT {{RESET-STREAM-AT}} to ensure
that stream prefix information is delivered when a WebTransport stream is reset
{{WEBTRANS-HTTP3}}.

This document does not change HTTP/3 semantics.  It describes cases where
existing HTTP/3 stream-error semantics can be combined with reliable delivery of
a bounded stream prefix.  The Reliable Size SHOULD identify the end of a
complete HTTP/3 frame.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Use Cases

## CONNECT Tunnels

The relay use case described in {{RESET-STREAM-AT}} directly applies to HTTP/3
CONNECT.  HTTP/3 CONNECT carries tunnel data in DATA frames after a successful
CONNECT response.  {{HTTP3}} says that TCP connection errors, including TCP
reset, are signaled as stream errors of type H3_CONNECT_ERROR.

A proxy might receive bytes from the upstream TCP connection and then observe an
upstream reset.  With a plain stream reset, DATA frames containing those
upstream bytes might be lost and will not be retransmitted.  Using
RESET_STREAM_AT allows the proxy to signal H3_CONNECT_ERROR while still
delivering the upstream bytes that it has committed to forwarding.

For this use case, the Reliable Size SHOULD be the end of the last committed
DATA frame carrying tunnel bytes from the upstream connection.

## Intermediaries After Response Commitment

An intermediary can usually translate an upstream failure into a new HTTP
response if it has not yet sent response headers downstream.  For example, it
can generate a 502 response.

Once response headers have been forwarded, that option is no longer available.
If the intermediary later detects an upstream stream error, a malformed upstream
response, or another stream-local failure, it cannot replace the response with a
new status code.

RESET_STREAM_AT allows the intermediary to preserve the response prefix that it
already committed while still terminating the stream with an appropriate HTTP/3
error code.  The Reliable Size SHOULD be the end of the last complete HTTP/3
frame that the intermediary committed to forwarding downstream.

This use case is mostly diagnostic, but it can also be useful for clients that
consume meaningful prefixes of streaming responses.

## Malformed Decoded HTTP Messages

This use case applies after an HTTP/3 HEADERS frame has been parsed and QPACK
decompression has succeeded.  It does not apply to invalid HTTP/3 frame syntax,
invalid frame ordering, or QPACK decompression failure. {{HTTP3}} defines
malformed HTTP messages to include invalid pseudo-header fields, missing
mandatory pseudo-header fields, pseudo-header fields after regular fields,
prohibited fields, uppercase field names, invalid field names or values, and
Content-Length mismatches.  A detected malformed message is treated as a stream
error of type H3_MESSAGE_ERROR.  For a malformed request, a server can send an
HTTP response indicating the error before closing or resetting the stream.

RESET_STREAM_AT allows that response to be delivered while preserving the
H3_MESSAGE_ERROR signal.  The Reliable Size SHOULD be the end of the diagnostic
response header block, or the end of that header block plus a short diagnostic
response body.

This can help debug faulty client implementations, since the response can carry
more information than a stream reset error code alone.

## Requests Rejected Before Application Processing

{{HTTP3}} defines H3_REQUEST_REJECTED for requests that a server rejects without
performing application processing.  A client can treat such a request as though
it had never been sent.  H3_REQUEST_REJECTED is not used after partial or full
application processing.

A server might want to preserve this retry signal while also providing a small
diagnostic response.  For example, a server that is draining, overloaded, or
unable to route a request might send a 503 or 429 response with retry
information, then terminate the stream with H3_REQUEST_REJECTED.

With RESET_STREAM_AT, the Reliable Size SHOULD be the end of the diagnostic
response header block, or the end of that header block plus a short diagnostic
response body. This is mainly useful for HTTP/3-aware clients and debugging
tools.  A generic HTTP client API might not expose both the HTTP response and
the stream error.


# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
