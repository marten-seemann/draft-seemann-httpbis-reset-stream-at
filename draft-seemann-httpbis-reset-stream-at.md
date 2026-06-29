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
reset.  This is a problem for HTTP/3 when the correct protocol outcome is an
abrupt stream termination, but the sender still has a bounded prefix worth
delivering, such as response headers, a short diagnostic response body, or a
partial response received from an upstream host.

RESET_STREAM_AT provides reliable delivery of that prefix and is already used by
WebTransport over HTTP/3 when resetting streams.  The same mechanism also fits
HTTP/3 CONNECT tunnels, intermediary failures after response commitment,
malformed decoded HTTP messages, and requests rejected before application
processing.


--- middle

# Introduction

HTTP/3 {{HTTP3}} uses QUIC streams to carry requests and responses.  Several
HTTP/3 errors are signaled by abruptly terminating a stream with an error code,
but a QUIC stream reset does not ensure delivery of data sent before the reset.

This matters when the sender has a useful bounded prefix, but the correct HTTP/3
outcome is still a stream error.  Examples include a proxy that has received
response bytes before an upstream failure, or a server that has decoded a
malformed request and wants to send a short diagnostic response before using
H3_MESSAGE_ERROR.

RESET_STREAM_AT {{RESET-STREAM-AT}} adds a Reliable Size to the reset, so stream
data up to that point is delivered reliably while the stream still terminates
abruptly.  WebTransport over HTTP/3 {{WEBTRANS-HTTP3}} already uses this
extension to ensure delivery of the prefix that associates a WebTransport stream
with its session.

The following sections describe HTTP/3 cases where reliable prefix delivery is
useful while retaining the existing stream-error signal.

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

An intermediary can translate an upstream failure into a new HTTP response only
before it sends response headers downstream.  After that, it cannot replace the
response with a new status code.

If the intermediary later detects an upstream stream error, a malformed
upstream response, or another stream-local failure, RESET_STREAM_AT allows it to
terminate the stream with an appropriate HTTP/3 error code while making
reliable only the response portion it can safely identify.  That portion always
includes the committed response HEADERS frame, and can include later response
data when the intermediary understands enough of the application protocol to
choose a useful boundary.  The Reliable Size SHOULD therefore be the end of the
response HEADERS frame, or the end of a later complete HTTP/3 frame forwarded
downstream.

This is mainly useful for diagnostics, logging, and clients that can consume a
partial streaming response before an error.

## Malformed Decoded HTTP Messages

This use case applies after an HTTP/3 HEADERS frame has been parsed and QPACK
decompression has succeeded.  It does not apply to invalid HTTP/3 frame syntax,
invalid frame ordering, or QPACK decompression failure.  Examples of malformed
HTTP messages under {{HTTP3}} include invalid pseudo-header fields and uppercase
field names.  A detected malformed message is treated as a stream error of type
H3_MESSAGE_ERROR.

RESET_STREAM_AT allows a diagnostic HTTP response to be delivered while
preserving the H3_MESSAGE_ERROR signal.  The Reliable Size SHOULD be the end of
the diagnostic response HEADERS frame, or the end of a later complete DATA frame
carrying a short diagnostic response body.

The additional information can be valuable when debugging faulty or
non-conforming clients.

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
response HEADERS frame, or the end of a later complete DATA frame carrying a
short diagnostic response body.  This is mainly useful for HTTP/3-aware clients
and debugging tools.  A generic HTTP client API might not expose both the HTTP
response and the stream error.


# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
