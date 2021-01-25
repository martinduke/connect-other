---
title: The PortsOnly Header Field: Arbitrary Transports over CONNECT-UDP
abbrev: The PortsOnly Header Field
docname: draft-duke-masque-proto-number-00
category: exp

ipr: trust200902
keyword: Internet-Draft

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

This document describes a new HTTP header called PortsOnly. When used with
the CONNECT-UDP HTTP method, it enables a proxy to tunnel arbitrary transport
protocols over HTTP streams.

--- middle

# Introduction {#introduction}

The HTTP CONNECT method (section 4.3.6 of {{?RFC7231}}) has long provided a
means of tunneling a TCP connection over an HTTP stream. The CONNECT-UDP
method {{!I-D.ietf-masque-connect-udp}} extends this capability to include UDP
datagrams over a stream.

As the CONNECT-UDP method delivers discrete datagrams to each endpoint, it can
extend conceptually to any packetized protocol. The PortsOnly Header field
allows a CONNECT-UDP proxy to tunnel packets with non-TCP, non-UDP protocol
numbers, as long as the corresponding protocol meets minimal formatting
requirements.

Specifically, any protocol header where the first four octets encode the source
and destination ports can be tunneled using this framework. The client and proxy
include all other protocol header information in the datagrams delivered over
the tunnel.

The PortsOnly Header Field is defined in accordance with
{{!I-D.ietf-httpbis-header-structure}} to improve the safety of using and
processing this field.

## Conventions and Definitions {#conventions}

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

In this document, we use the term "proxy" to refer to the HTTP server that
opens the UDP socket and responds to the CONNECT-UDP request. If there are
HTTP intermediaries (as defined in Section 2.3 of {{RFC7230}}) between the
client and the proxy, those are referred to as "intermediaries" in this
document.

# The PortsOnly Header Field

PortsOnly is a request and response Structured Header field that modifies
CONNECT-UDP so that the proxy only appends the first four octets of a UDP header
(the Source and Destination Ports) to datagrams from the client, and uses an IP 
header with a protocol number (IPv4) or Next Header (IPv6) set to the value in
the field.

Similarly, the proxy will deliver packets from the server to the stream if and
only if the protocol number or next header matches the specified value, rather
than the value for UDP (17), in addition to the ports relevant to that stream.

The field value is an Item of type Integer. It has no parameters.

The ABNF is:
PortsOnly = sf-integer

The field value MUST have a value between 0 and 255, inclusive. Any other
value means the header is malformed.

An endpoint MUST ignore a PortsOnly field in any method other than
CONNECT-UDP.

## Sending PortsOnly

When a client sends the PortsOnly header field, it MUST use a value that
corresponds to a protocol whose first four octets of each packet correspond to
the source and destination ports. For example, 33 (DCCP, {{?RFC4330}}); 132
(SCTP, {{?RFC4960}}); and 136 (UDPLite, {{?RFC3828}}) would all be valid
choices.

Subsequent datagrams, whether sent in DATAGRAM frames or Stream Chunks, MUST
include all parts of the transport protocol header except for the port
encodings.

Unlike CONNECT-UDP clients that do not use PortsOnly, clients MUST NOT not send
datagrams or stream chunks optimistically, as the server might not successfully
process the PortsOnly header field and treat them as UDP datagrams, unless it
has received a CONNECT_UDP_PORTS_ONLY setting from the proxy over an HTTP/2 or
HTTP/3 connection.

If the response does not include a PortsOnly header with the same field value
as the request, the client MUST abort the stream.

DATAGRAM frames or Stream Chunks arriving from the proxy will not contain the
port fields of the header, but can otherwise be processed as a datagram of that
protocol that arrived over IP.

## Proxy Processing of PortsOnly

A proxy that supports the PortsOnly header field SHOULD include the
CONNECT_UDP_PORTS_ONLY setting in its HTTP/2 or HTTP/3 SETTINGS frame to
reduce client latency.

A proxy MUST respond to a PortsOnly header with an PortsOnly header with an
identical field value.

The proxy assigns a source and destination ports identically to conventional
CONNECT-UDP and prepends these to datagrams from the client, in addition to an
IP header with the field value in the Protocol Number (IPv4) or Next Header
(IPv6) field.

The proxy matches packets arriving from the server to a CONNECT-UDP stream if
the Protocol Number or Next Header field matches, and the first four octets
after the IP header match the assigned ports.

## HTTP Intermediaries

HTTP Intermediaries that discover that an upstream proxy does not support
the PortsOnly header MUST abort the stream in the direction of the client.

# Security Considerations {#security}

CONNECT-UDP streams that use the PortsOnly header have similar security
properties to other CONNECT-UDP streams, as described in
{{!I-D.ietf-masque-connect-udp}}.

However, as more of the transport header originates at the server, and the
tunneled protocols are less ubiquitous than UDP, these headers may serve to
fingerprint the protocol implementation that generated them.

Furthermore, more degrees of freedom enhances the ability of clients to
induce the proxy to generate certain packets, which can have undesirable
effects in the network while being less traceable to the client.

# IANA Considerations

## PortsOnly 

This document defines the "PortsOnly" HTTP header field, and registers it in the
Permanent Message Header Fields registry.

o Header field name: PortsOnly
o Applicable Protocol: HTTP
o Status: experimental
o Author/Change controller: IETF
o Specification document(s): This document
o Related information: for other transports over CONNECT-UDP

## CONNECT_UDP_PORTS_ONLY Settings Frame

The document adds this entry to both the HTTP/2 Settings and HTTP/3 Settings
registry.

o Code: TBD
o Name: CONNECT_UDP_PORTS_ONLY
o Initial Value: 0
o Reference: This document

--- back

# Acknowledgments {#acknowledgments}
{:numbered="false"}
