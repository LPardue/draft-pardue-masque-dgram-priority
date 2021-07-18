---
title: HTTP Datagram Prioritization
abbrev: HTTP Datagram Prioritization
docname: draft-pardue-masque-dgram-priority-latest
category: exp

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
{{!QUIC-DATAGRAM=I-D.ietf-quic-datagram}}, to carry application data. Streams and
datagrams can be multiplexed but QUIC provides no interoperable prioritization
scheme or signaling mechanism itself. The HTTP Extensible Prioritization scheme
{{!I-D.ietf-httpbis-priority}} describes how to prioritize streams in HTTP/2 and
HTTP/3. This document adopts the scheme to support HTTP datagrams
{{!HTTP-DATAGRAM=I-D.ietf-masque-h3-datagram}}.

The Extensible Priorities scheme for HTTP describes how clients can send
priority signals related to requests in order to suggest how a server allocates
resources to serving responses. When the protocol is HTTP/2, responses are
carried on streams. When the protocol is HTTP/3, responses are carries on QUIC
streams.

While QUIC streams support multiplexing natively via use of a stream identifier,
the QUIC DATAGRAM extension does not provide any such identifier. HTTP datagrams
{{HTTP-DATAGRAM}} supports multiplexting using a set of application-level
identifiers that can be controlled and accessed by HTTP/3. One identifer relates
to a request stream, the second, optional, identifer relates to an abstract
context. {{HTTP-DATAGRAM}} does not, however, define any means for multiplexed
datagram prioritization.

When the application protocol is HTTP/3, HTTP Datagrams can map directly to QUIC
datagrams or they can be carried on streams using a DATAGRAM Capsule; see
{{Section 4.4 of HTTP-DATAGRAM}}.

This document describes how the Extensible Priorities scheme applies to HTTP
datagrams. Priority signals sent by clients, related to requests, can also be
considered input to server scheduling decisions for HTTP datagrams mapped to
QUIC datagrams.


## Notational Conventions

{::boilerplate bcp14}

The term sf-integer is imported from {{!STRUCTURED-FIELDS=RFC8941}}.

# Signalling Datagram Priority

The Extensible Prioritization scheme {{!I-D.ietf-httpbis-priority}} provides a
framework for communicating and acting upon priority parameters, using
{{STRUCTURED-FIELDS}} formats. It defines the urgency and incremental parameters
and provides guidance to implementers about how to act on these parameters, in
combination with other inputs, to make resource allocation and scheduling
choices. Urgency communicates the client-view of request importance, and
incremental communicates how the client intends to process response data as it
arrives. Parameters are communicated in HTTP headers or version-specific frames.
A client omitting the urgency or incremetal parameters can be interprested by
the server as a signal to apply default priorities. The core scheme is
extensible, new parameters can be defined to augment the base ones.

This specification defines the datagram-urgency (`du`) extension parameter that
operates in addition to the base urgency. There is no extension to the base
incremental behavior; individual datragrams, even if belonging to the same
identifier, are messages that are expected to be processed individually as they
arrive.

## Datagram Urgency

The datagram-urgency parameter (`du`) takes an integer between 0 and 7, in
descending order of priority. This range matches the base urgency (`u`)
paramenter range; see {{Section 4.1 of !I-D.ietf-httpbis-priority}}.

The value is encoded as an sf-integer. There is no default value.

This parameter indicates the sender's recommendation, based on the expectation
that the server would transmit HTTP datagrams in the order of their
datagram-urgency values if possible. The smaller the value, the higher the
precedence. Omitting the datagram-urgency parameter is a signal to apply the
value of the urgency parameter.

The following example shows a request for a CSS file with the urgency set to
`0`, any associated datagrams have the lower urgency of `2`:

~~~ example
:method = GET
:scheme = https
:authority = example.net
:path = /style.css
priority = u=0, du=2
~~~

Endpoints MUST NOT treat reception of the datagram-urgency parameter, even if
HTTP datagram support is not enabled.

The datagram-urgency parameter applies only to HTTP datagrams mapped to QUIC
datagrams. Datagram capsules are sent on streams, so the base urgency parameter
applies to them.

## Prioritization of Contexts

The datagram-urgency parameter applies to all HTTP datagram contexts related to
a request stream. Prioritization of individual contexts is not supported.

## Reprioritization

Reprioritization is supported using the existing mechanisms defined in {{Section
6 of !I-D.ietf-httpbis-priority}}.

# Client Scheduling

Clients MAY use datagram-urgency to make local processing or scheduling choices
about HTTP datagrams related to the requests it initiates.

# Server Scheduling

Priority signals are input to a prioritization process. Expressing priority is
only a suggestion. The datagram-urgency parameter introduces new scheduling
considerations on top of those presented in {{Section 10 of
!I-D.ietf-httpbis-priority}}.

It is RECOMMENDED that, when possible, servers send higher urgency HTTP
datagrams before lower urgency datagrams.

Where streams and datagrams have equal urgency and datagram-urgency, it is
RECOMMENDED that servers alternate emitting HTTP datagrams and stream bytes.
Where servers implement the recommendations in {{Section 10 of
!I-D.ietf-httpbis-priority}}, alternating between datagram and stream data will
result in fair scheduling. This recommendation holds whether stream are
incremental or not.

It is RECOMMENDED that servers schedule DATAGRAM capsules the same as response
data.

# Retranmission Scheduling

{{Section 12 of !I-D.ietf-httpbis-priority}} provides guidance about scheduling
of retransmission data vs. new data. Since QUIC datagrams are not retransmitted,
endpoints that prioritize QUIC stream retransmission data could delay datagrams.
Furthermore, since DATAGRAM capsules are sent as stream data, they **are** subject
to retransmission and could also delay native QUIC datagrams.

# Security Considerations

There are believed to be no additional considerations to those presented in
{{!I-D.ietf-httpbis-priority}}.

# IANA Considerations

This specification registers the following entry in the HTTP Priority Parameters
Registry

Name:
: datagram-urgency

Description:
: Priority of HTTP datagrams

Reference:
: This document

--- back

# Acknowledgements

This document is inspired by discussion by many people across HTTP, QUIC and
MASQUE WGs.

