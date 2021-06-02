---
title: "The Other-Transport Extension: Arbitrary Transports over CONNECT-UDP"
abbrev: The Other-Transport Extension
docname: draft-duke-masque-other-transport-latest
category: exp
ipr: trust200902
keyword: Internet-Draft
area: Transport
workgroup: MASQUE

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: "M. Duke"
    name: "Martin Duke"
    organization: "F5, Inc."
    street: "801 5th Ave"
    city: "Seattle, Washington, 98104"
    country: "United States of America"
    email: martin.h.duke@gmail.com


--- abstract

This document describes an extension to the HTTP CONNECT-UDP method
{{!CONNECTUDP=I-D.ietf-masque-connect-udp}} that supports tunneling of other
transport protocols, as long as the first four octets of those protocols encode
the source and destination ports.

--- middle

# Introduction {#introduction}

The HTTP CONNECT method (section 4.3.6 of {{?RFC7231}}) has long provided a
means of tunneling a TCP connection over an HTTP stream. The CONNECT-UDP
method {{!CONNECTUDP}} extends this capability to include UDP datagrams over a
stream.

As CONNECT-UDP delivers discrete datagrams to each endpoint, it can extend
conceptually to any packetized protocol. The Other-Transport extension allows a
CONNECT-UDP proxy to tunnel packets with non-TCP, non-UDP protocol numbers, as
long as the corresponding protocol meets minimal formatting requirements.

Specifically, any protocol header where the first four octets encode the source
and destination ports can be tunneled using this framework. The client and proxy
include all other protocol header information in the datagrams delivered over
the tunnel. For example, 33 (DCCP, {{?RFC4330}}); 132 (SCTP, {{?RFC4960}}); and
136 (UDPLite, {{?RFC3828}}) would all be valid candidates for Other-Transport.

In principle, TCP can be proxied using this extension as well. This might
provide advantages over traditional HTTP CONNECT if the client's TCP
implementation has features lacking at the proxy. 

## Conventions and Definitions {#conventions}

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

In this document, we use the term "proxy" to refer to the HTTP server that
opens the UDP socket and responds to the CONNECT-UDP request. If there are
HTTP intermediaries (as defined in Section 2.3 of {{?RFC7230}}) between the
client and the proxy, those are referred to as "intermediaries" in this
document.

# Other-Transport Header Definition

"Other-Transport" is an Item Structured Header
{{!HDRSTRUCT=I-D.ietf-httpbis-header-structure}}. Its ABNF is:

~~~
Other-Transport = sf-integer
~~~

The value MUST be between 0 and 255, inclusive. Any other value is malformed.
This value indicates the value of the Protocol Number (IPv4) or Next Header
(IPv6) in IP headers corresponding to the CONNECT-UDP stream.

An Other-Transport header is ignored in any method other than CONNECT-UDP.

A client that sends this header MUST include the entire transport header, with
the exception of the first four octets, in each HTTP/3 DATAGRAM or Stream Chunk
payload it sends. It MUST process incoming datagrams with the same assumption.

When a client sends the Other-Transport header field, it MUST use a value that
corresponds to a protocol whose first four octets of each packet correspond to
the source and destination ports. For example, 33 (DCCP, {{?RFC4330}}); 132
(SCTP, {{?RFC4960}}); and 136 (UDPLite, {{?RFC3828}}) would all be valid
choices.

A proxy MUST NOT include this header unless it will prepend and strip port
numbers instead of entire UDP headers, and use the Protocol Number in IP headers
for both packets to the server and for demultiplexing incoming server packets.

A proxy MAY choose not to send the header due to policy regarding specific
protocol numbers.

This extension is said to have been negotiated when both client and proxy
indicate support for it in their CONNECT-UDP request and response using the
same value.

If the server responds without the Other-Transport header, the client may either
proceed with UDP datagrams or close the stream.

A response with a Other-Transport value other than that provided by the client
is malformed.

# Datagram Encoding of Proxied Packets

All DATAGRAM frames corresponding to the negotiated Datagram-Flow-Id headers
are processed in accordance with the Other-Transport extension, if negotiated.

All DATAGRAM frames MUST include the entire IP payload with the exception of the
first four octets.

If a client sent an Other-Transport header, it MUST NOT send DATAGRAM frames
until it confirms this extension has been negotiated. If the proxy does not
support Other-Transport, it will interpret DATAGRAM frames as UDP payloads, with
unpredictable results.

# Stream Encoding of Proxied Packets

Endpoints use the 0x10 Stream Chunk Type to encode datagrams.

Clients MAY send payloads using Stream Chunks before negotiation is complete.
Proxies that do not support the extension will simply ignore these chunks.

# HTTP Intermediaries

HTTP Intermediaries that discover that an upstream proxy does not support
the Other-Transport header MUST abort the stream in the direction of the client.

# Security Considerations {#security}

CONNECT-UDP streams that use the Other-Transport header have similar security
properties to other CONNECT-UDP streams, as described in
{{!CONNECTUDP}}.

However, as more of the transport header originates at the server, and the
tunneled protocols are less ubiquitous than UDP, these headers may serve to
fingerprint the protocol implementation that generated them.

Furthermore, additional control over packet headers enhances the ability of
clients to induce the proxy to generate certain packets, which might have
undesirable effects in the network while being less traceable to the client.

# IANA Considerations

## HTTP Header

This document requests that IANA registers the "Other-Transport" header in the
"Permanent Message Header Field Names" registry maintained at
<[](https://www.iana.org/assignments/message-headers)>.

~~~
  +-------------------+----------+--------+---------------+
  | Header Field Name | Protocol | Status |   Reference   |
  +-------------------+----------+--------+---------------+
  |   Other-Transport |   http   |  std   | This document |
  +-------------------+----------+--------+---------------+
~~~


## Stream Chunk Type Registration {#iana-chunk-type}

This document will request IANA to register the following entry in the
"CONNECT-UDP Stream Chunk Type" registry {{CONNECTUDP}}:

~~~
  +-------+-----------------+-------------------------+---------------+
  | Value |      Type       |       Description       |   Reference   |
  +-------+-----------------+-------------------------+---------------+
  | 0x10  | OTHER_TRANSPORT | Other Transport Protocol| This document |
  +-------+-----------------+-------------------------+---------------+
~~~

--- back

# Acknowledgments {#acknowledgments}
{:numbered="false"}
