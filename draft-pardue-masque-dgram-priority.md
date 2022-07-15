---
title: HTTP Datagrams, UDP Proxying, and Extensible Prioritization
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

--- note_Note_to_Readers

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
directly to QUIC datagrams, in which case they carry a Quarter Stream ID - an
encoding of the request stream ID - that is used to demultiplex at the receiver;
see {{Section 3.1 of HTTP-DATAGRAM}}. {{HTTP-DATAGRAM}} also defines the
DATAGRAM capsule, which can be used for reliable delivery over all versions of
HTTP; see {{Section 3.5 of HTTP-DATAGRAM}}. In all cases, the prioritization of
datagrams is noted as unspecified and delegated to future extensions.

This document describes how the Extensible Priorities scheme can be augmented to
also apply to HTTP datagrams that are multiplexed with other flows or streams.
It enhances the Priority signals sent by clients, with a new datagram-urgency
(`du`) parameter ({{datagram-urgency}}) and explains how this input is to be
considered in server scheduling decisions for HTTP datagrams mapped to QUIC
datagrams; see {{server-scheduling}}.

When HTTP datagrams are used for proxying UDP, additional use cases extending
beyond UDP data transfer are supported by the use of context IDs; see {{Section
4 of !HTTP-UDP-PROXY=I-D.ietf-masque-connect-udp}}. The PRIORITY_UPDATE capsule
type can be used to signal the datagram-priority of individual contexts; see
{{capsule}}.


## Notational Conventions

{::boilerplate bcp14}

The term Integer is imported from {{!STRUCTURED-FIELDS=RFC8941}}.

# Signalling Datagram Priority

The Extensible Prioritization scheme {{PRIORITY}} describes how clients can
send priority signals related to requests. Signals are a set
of parameters, encoded using {{STRUCTURED-FIELDS}}.

{{PRIORITY}} defines the urgency and incremental parameters and provides
guidance about how implementers can act on these parameters, in combination
with other inputs, to make resource allocation and scheduling choices. Urgency
communicates the client-view of request importance, and incremental communicates
how the client intends to process response data as it arrives. Parameters are
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

The datagram-urgency (`du`) parameter is Integer (see {{Section 3.3.1 of
STRUCTURED-FIELDS}}), between 0 and 7, in descending order of priority. This
range matches the base urgency (`u`) parameter range; see Section 4.1 of
{{PRIORITY}}. However, there is no default value.

This parameter indicates the sender's recommendation, based on the expectation
that the server would transmit HTTP datagrams in the order of their
datagram-urgency values if possible. The smaller the value, the higher the
precedence. Omitting the datagram-urgency parameter is a signal to apply the
value of the urgency parameter.

The following example shows a request for a CSS file with the urgency set to
`0`, any associated datagrams have the lower datagram-urgency of `2`:

~~~ example
:method = GET
:scheme = https
:authority = example.net
:path = /style.css
priority = u=0, du=2
~~~

Note that when the urgency parameter is omitted, it's default value of `3` is
applied. In the following example, the priority field is omitted entirely,
invoking the default behaviour of urgency and datagram-urgency, causing them
to both have the implicit value `3`:

~~~ example
:method = GET
:scheme = https
:authority = example.net
:path = /style.css
~~~

Endpoints MUST NOT treat reception of the datagram-urgency parameter as an
error, even if HTTP datagram support is not enabled.

The datagram-urgency parameter applies only to HTTP datagrams mapped to QUIC
datagrams. Datagram capsules are sent on streams, so the base urgency parameter
applies to them.

# Reprioritization

Reprioritization behaves similarly to existing mechanisms defined in {{Section 6
of PRIORITY}}. PRIORITY_UPDATE frames can be sent by clients to provide updated
priority signals after the initial request has been sent.

# Prioritization when Proxying UDP in HTTP

{{HTTP-UDP-PROXY}} describes how to proxy UDP using HTTP datagrams. Client make
UDP proxying requests using Extended CONNECT, which initiates a UDP tunnel. HTTP
datagrams related to this stream correspond to the UDP tunnel by default. In
order support extension use cases, {{Section 4 of HTTP-UDP-PROXY}} defines
context IDs, that are sent within datagrams, in addition to the Quarter Stream
ID. UDP payloads use context ID 0, forms of data use other IDs.

Datagram priority applies to UDP proxying requests, as described in
{{datagram-urgency}}. By default the same datagram-urgency applies to all HTTP
datagram contexts related to the request stream.

## The PRIORITY_UPDATE Capsule {#capsule}

There might be cases where it is beneficial to prioritize individual contexts
differently from one another. This document defines the PRIORITY_UPDATE (TBD)
capsule type to carry a priority signal related to individual contexts.

Once a UDP proxy request converts to the capsule protocol (see {{Section 3 of
HTTP-UDP-PROXY}}, clients can send PRIORITY_UPDATE capsules to signal the
priority of the identified context.

A PRIORITY_UPDATE capsule communicates a complete set of all priority parameters
in the Priority Field Value field. Omitting a priority parameter is a signal to
derive a value from defaults; see {{datagram-urgency}}. Failure to parse the
Priority Field Value MAY be treated as a connection error. In HTTP/2, the error
is of type PROTOCOL_ERROR; in HTTP/3, the error is of type
H3_GENERAL_PROTOCOL_ERROR.

TODO: describe what happens if capsules arrive before contexts exists. Buffer?
Drop?

TODO: consider if servers could send this capsule type

~~~
Priority_Update Capsule {
    Type (i) = PRIORITY_UPDATE,
    Length (i),
    Context ID (i),
    Priority Field Value (..),
}
~~~
{: #fig-priority_update capsule title="PRIORITY_UPDATE Capsule Format"}

The PRIORITY_UPDATE capsule has the following fields:

Context ID:
: The context ID that is the target of the priority update.

Priority Field Value:
: The priority update value in ASCII text, encoded using Structured Fields; see
 {{PRIORITY}}.


# Client Scheduling

A client MAY use datagram-urgency to make local processing or scheduling choices
about HTTP datagrams related to the requests it initiates.

# Server Scheduling

Priority signals are input to a prioritization process. Expressing priority is
only a suggestion. The datagram-urgency parameter introduces new scheduling
considerations on top of those presented in {{Section 10 of
PRIORITY}}.

It is RECOMMENDED that, when possible, servers respect the datagram-urgency
parameter, sending higher-urgency HTTP datagrams before lower-urgency datagrams.

Where streams and datagrams have equal urgency and datagram-urgency
respectively, a server needs to decide how to divide the available sending
capacity between stream and datagram data. Strict or static preference for one
type of data over another (e.g., datagrams first, then streams) could lead to
suboptimal results at the client, depending on the nature of the data. This is a
form of starvation, as defined in {{Section 10 of PRIORITY}}. It applies whether
the streams are incremental or not.

Similarly, if datagrams are used for HTTP proxying and there are multiple
context IDs in use for different purposes, those purposes might interfere or
starve each other if they have the equal datagram-urgency.

It is RECOMMENDED that servers avoid such starvation where possible. The method
for doing so is an implementation decision. One approach is to divide the
available bandwidth between stream and datagram data in some fixed or
dynamic ratio. For instance, a server could choose to generate two classes of
application data QUIC packets: STREAM-frame-only packets and DATAGRAM-only-frame
packets.  The server can control the capacity ratio split by managing the
frequency of the packet classes. A simple alternating strategy would result in a
roughly 50/50 split, while other frequencies would produce different ratios.

When HTTP datagrams are carried in DATAGRAM capsules. It is RECOMMENDED that
servers schedule the capsules in the manner expected for response data; see
{{Section 10 of PRIORITY}}.


# Retransmission Scheduling

{{Section 12 of PRIORITY}} provides guidance about scheduling of retransmission
data vs. new data. Since QUIC datagrams are not retransmitted, endpoints that
prioritize QUIC stream retransmission data could delay datagrams. Furthermore,
since DATAGRAM capsules are sent as stream data, they **are** subject to
retransmission and could also delay native QUIC datagrams.

# Security Considerations

There are believed to be no additional considerations to those presented in
{{PRIORITY}}.

# IANA Considerations

This specification registers the following entry in the HTTP Priority Parameters
Registry

Name:
: du

Description:
: The urgency of HTTP datagrams associated with a response.

Reference:
: This document

--- back

# Acknowledgements

This document is inspired by discussion by many people across HTTP, QUIC and
MASQUE WGs.

