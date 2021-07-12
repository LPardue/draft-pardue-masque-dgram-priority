---
title: HTTP Datagram Prioritization
abbrev: HTTP Datagram Prioritization
docname: draft-pardue-masque-dgram-priority-latest
category: experimental

ipr: trust200902
area: Transport
workgroup: MASQUE
keyword: Internet-Draft

stand_alone: yes
smart_quotes: no
pi: [toc, docindent, sortrefs, symrefs, strict, compact, comments, inline]

author:
    ins: L. Pardue
    name: Lucas Pardue
    org: Cloudflare
    email: lucaspardue.24.7@gmail.com

normative:

--- abstract

Application protocols using the QUIC transport protocol rely on streams, and
optionally the DATAGRAM extension, to carry application data. Streams and
datagrams can be multiplexed but QUIC provides no interoperable prioritization
scheme or signaling mechanism itself. The HTTP Extensible Prioritization scheme
describes how to prioritize streams in HTTP/2 and HTTP/3. This document adopts
the scheme to support HTTP datagrams.

--- note_Note_tho_Readers

*RFC EDITOR: please remove this section before publication*

Source code and issues list for this draft can be found at
<https://github.com/LPardue/draft-pardue-masque-dgram-priority>.

--- middle

# Introduction

Application protocols using the QUIC transport protocol {{?QUIC=RFC9000}} rely
on streams, and optionally the DATAGRAM extension
{{!DATAGRAM=I-D.ietf-quic-datagram}}, to carry application data. Streams and
datagrams can be multiplexed but QUIC provides no interoperable prioritization
scheme or signaling mechanism itself. The HTTP Extensible Prioritization scheme
{{!I-D.ietf-httpbis-priority}} describes how to prioritize streams in HTTP/2 and
HTTP/3. This document adopts the scheme to support HTTP datagrams
{{!I-D.ietf-masque-h3-datagram}}.

The Extensible Priorities scheme for HTTP describes how clients can send
priority signals related to requests in order to suggest how a server allocates
resources to serving responses. When the protocol is HTTP/2, responses are
carried on streams. When the protocol is HTTP/3, responses are carries on QUIC
streams.

While QUIC streams support multiplexing natively via use of a stream identifier,
the QUIC DATAGRAM extension does not provide any such identifier.
{{!I-D.ietf-masque-h3-datagram}} defines a set of identifiers that can be
controlled and accessed by HTTP. When the protocol is HTTP/2 or HTTP/3, one
identifer relates to a request stream. A second, optional, identifer relates to
an abstract context. {{!I-D.ietf-masque-h3-datagram}} does not define any means
for multiplexed datagram prioritization.

This document describes how the Extensible Priorities scheme applies to HTTP
datagrams. Signals sent by clients related to requests can also be considered
input to server scheduling decisions regarding HTTP datagrams.



## Notational Conventions

{::boilerplate bcp14}

# Signalling Datagram Priority

The Extensible Prioritization scheme {{!I-D.ietf-httpbis-priority}} provides a
framework for communicating and acting upon priority parameters. It defines the
urgency and incremental parameters and provides guidance to implementers about
how to act on these parameters, in combination with other inputs, to make
resource allocation and scheduling choices. Urgency communicates the client-view
of request importance and incremental communicates how the client intends to
process response data as it arrives. Parameters are communicated in HTTP headers
or version-specific frames. Omitting a priority signal indicates to the server
to apply default priorities. The core scheme is extensible, new parameters can
be defined to augment the base ones.

## Datagram Urgency

The datagram-urgency parameter (`du`) takes an integer between 0 and 7, in
descending order of priority. This range matches the base urgency (`u`)
paramenter range.

The value is encoded as an sf-integer. The default value is 3.

This parameter indicates the sender's recommendation, based on the expectation
that the server would transmit HTTP datagrams in the order of their urgency
values if possible. The smaller the value, the higher the precedence.

The following example shows a request for a CSS file with the urgency set to
`0`, any associated datagrams will be delivered with the lower urgency of `2`:

~~~ example
:method = GET
:scheme = https
:authority = example.net
:path = /style.css
priority = u=0, du=2
~~~

# Scheduling guidance

TBD - bikeshed

# Security Considerations

TBD

# IANA Considerations

TBD

--- back

# Acknowledgements

This document is inspired by discussion by many people across HTTP, QUIC and
MASQUE WGs.

