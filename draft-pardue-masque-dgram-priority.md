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
optionally the unreliable datagram extension, to carry application data. Streams
and datagrams can be multiplexed in single connections but QUIC does not define
an interoperable prioritization scheme or signaling mechanism. The HTTP
Extensible Prioritization scheme describes an application-level scheme for the
prioritization of streams in HTTP/2 and HTTP/3. This document defines how
Extensible Priorities can be augmented to apply to the multiplexing of HTTP
datagram flows with other flows or streams.

--- note_Note_tho_Readers

*RFC EDITOR: please remove this section before publication*

Source code and issues list for this draft can be found at
<https://github.com/LPardue/draft-pardue-masque-dgram-priority>.

--- middle

# Introduction

Application protocols using the QUIC transport protocol {{?QUIC=RFC9000}} rely
on streams, and optionally the unreliable datagram extension
{{!QUIC-DATAGRAM=RFC9221}}, to carry application data. Streams and datagrams can
be multiplexed in single connections but QUIC does not define an interoperable
prioritization scheme or signaling mechanism. The HTTP Extensible Prioritization
scheme {{!PRIORITY=RFC9218}} describes an application-level scheme for the
prioritization of streams in HTTP/2 and HTTP/3. This document defines how
Extensible Priorities can be applied to the multiplexing of HTTP datagram
{{!HTTP-DATAGRAM=I-D.ietf-masque-h3-datagram}} flows with other flows or
streams.

The Extensible Priorities scheme for HTTP describes how clients can send
priority signals related to requests in order to suggest how a server allocates
resources to serving responses. When the protocol is HTTP/2, responses are
carried on streams. When the protocol is HTTP/3, responses are carries on QUIC
streams.

While QUIC streams support multiplexing natively via use of a stream identifier,
the unreliable datagram extension does not provide any such multiplexing
identifier.

HTTP datagrams ({{HTTP-DATAGRAM}}) defines how multiplexed, potentially
unreliable datagrams can be sent inside an HTTP connection. All datagrams are
always associated with a request stream. In HTTP/3, HTTP datagrams can map
directly to QUIC datagrams, in which case they carry an encoding of the stream
ID that is used to demultiplex at the receiver; see {{Section 3.1 of
HTTP-DATAGRAM}}. {{HTTP-DATAGRAM}} also defines the DATAGRAM capsule, which can
be used for reliable delivery over all versions of HTTP; see
{{Section 3.5 of HTTP-DATAGRAM}}. In all cases, the prioritization of datagrams
is noted as unspecified and delegated to future extensions.

This document describes how the Extensible Priorities scheme can be augmented to
also apply to HTTP datagrams that are multiplexed with other flows or streams.
It enhances the Priority signals sent by clients, with a new `du` parameter and
explains how this input is to be considered in server scheduling decisions
for HTTP datagrams mapped to QUIC datagrams.


## Notational Conventions

{::boilerplate bcp14}

The term Integer is imported from {{!STRUCTURED-FIELDS=RFC8941}}.

# Signalling Datagram Priority

The Extensible Prioritization scheme {{!PRIORITY}} provides a framework for
communicating and acting upon priority parameters, using {{STRUCTURED-FIELDS}}
formats. It defines the urgency and incremental parameters and provides guidance
to implementers about how to act on these parameters, in combination with other
inputs, to make resource allocation and scheduling choices. Urgency communicates
the client-view of request importance, and incremental communicates how the
client intends to process response data as it arrives. Parameters are
communicated in HTTP headers or version-specific frames. A client omitting the
urgency or incremental parameters can be interpreted by the server as a signal
to apply default priorities. The core scheme is extensible, new parameters can
be defined to augment the base ones.

This specification defines the datagram-urgency (`du`) extension parameter that
operates in addition to the base urgency. There is no extension to the base
incremental behavior; individual datagrams, even if belonging to the same
identifier, are messages that are expected to be processed individually as they
arrive.

## Datagram Urgency

The datagram-urgency parameter (`du`) is Integer (see Section 3.3.1 of
{{STRUCTURED-FIELDS}}), between 0 and 7, in descending order of priority. This
range matches the base urgency (`u`) parameter range; see Section 4.1 of
{{!PRIORITY}}. However, there is no default value.

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

Endpoints MUST NOT treat reception of the datagram-urgency parameter as an
error, even if HTTP datagram support is not enabled.

The datagram-urgency parameter applies only to HTTP datagrams mapped to QUIC
datagrams. Datagram capsules are sent on streams, so the base urgency parameter
applies to them.

## Prioritization of Contexts

The datagram-urgency parameter applies to all HTTP datagram contexts related to
a request stream. Prioritization of individual contexts is not supported.

## Reprioritization

Reprioritization is supported using the existing mechanisms defined in {{Section
6 of PRIORITY}}.

# Client Scheduling

Clients MAY use datagram-urgency to make local processing or scheduling choices
about HTTP datagrams related to the requests it initiates.

# Server Scheduling

Priority signals are input to a prioritization process. Expressing priority is
only a suggestion. The datagram-urgency parameter introduces new scheduling
considerations on top of those presented in {{Section 10 of
PRIORITY}}.

It is RECOMMENDED that, when possible, servers send higher urgency HTTP
datagrams before lower urgency datagrams.

Where streams and datagrams have equal urgency and datagram-urgency, it is
RECOMMENDED that servers alternate emitting HTTP datagrams and stream bytes.
Where servers implement the recommendations in {{Section 10 of
PRIORITY}}, alternating between datagram and stream data will
result in fair scheduling. This recommendation holds whether stream are
incremental or not.

It is RECOMMENDED that servers schedule DATAGRAM capsules the same as response
data.

# Retransmission Scheduling

{{Section 12 of PRIORITY}} provides guidance about scheduling
of retransmission data vs. new data. Since QUIC datagrams are not retransmitted,
endpoints that prioritize QUIC stream retransmission data could delay datagrams.
Furthermore, since DATAGRAM capsules are sent as stream data, they **are**
subject to retransmission and could also delay native QUIC datagrams.

# Security Considerations

There are believed to be no additional considerations to those presented in
{{!PRIORITY}}.

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

