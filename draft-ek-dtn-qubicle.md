---
title: DTN QUIC Bundle Protocol Convergence Layer (qubicle)
abbrev: DTN QUIC CL
category: std

docname: draft-ek-dtn-qubicle-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: INT
workgroup: Delay/Disruption Tolerant Networking

keyword:

- DTN
- BPv7
- QUIC
- Convergence Layer

venue:
  group: Delay/Disruption Tolerant Networking
  type: Working Group
  mail: dtn@ietf.org
  arch: https://mailarchive.ietf.org/arch/browse/dtn/
  github: ekline/draft-dtn-qubicle
  latest: https://ekline.github.io/draft-dtn-qubicle/draft-ek-dtn-qubicle.html

author:
- fullname: Rick Taylor
  organization: Aalyria Technologies
  email: rtaylor@aalyria.com
- fullname: Erik Kline
  organization: Aalyria Technologies
  email: ek.ietf@gmail.com

normative:
  AttrLeaf: RFC8552
  BTP-U: I-D.ietf-dtn-btpu

informative:
  RFC9308:

--- abstract

This document specifies a minimal convergence layer protocol for transferring Bundle Protocol version 7 (BPv7) bundles over QUIC. The protocol leverages QUIC's native capabilities for reliable streaming, connection management, and security. Reliable transfers use QUIC unidirectional streams carrying CBOR Sequences of definite-length byte strings, each containing exactly one bundle. A stream can carry successive bundles without closing between them. Unreliable transfers use the Bundle Transfer Protocol - Unidirectional (BTP-U) over QUIC datagrams.

--- middle

# Introduction

Bundle Protocol version 7 (BPv7) {{!RFC9171}} requires Convergence Layer
Adapters (CLAs) to transfer bundles between nodes. This document specifies
the QUIC Bundle Protocol Convergence Layer, referred to in this document as
"Qubicle", a minimal CLA using QUIC {{!RFC9000}} that embraces QUIC's native
capabilities rather than layering additional protocol machinery.

The design philosophy is simple: QUIC already provides reliable streams, multiplexing, flow control, congestion control, and integrated security. This specification adds only what is strictly necessary to transfer bundles.

The protocol provides two services:

Reliable Service:
: Bundles are transferred using reliable, ordered QUIC stream transport,
  with successive bundles permitted on each unidirectional stream. Each
  bundle is wrapped in a definite-length CBOR byte string. This does not
  guarantee successful completion across arbitrary contact loss or receiver
  failure.

Unreliable Service:
: Bundles are transferred via QUIC datagrams {{!RFC9221}} using {{BTP-U}} framing.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

BPA:
: Bundle Protocol Agent, as defined in {{!RFC9171}}.

CLA:
: Convergence Layer Adapter, as defined in {{!RFC9171}}. Also abbreviated
  "CL" where the context is clear.

Client:
: The Qubicle peer that initiates the QUIC connection. This is a connection-level role and does not imply any restriction on bundle transfer direction.

Server:
: The Qubicle peer that accepts the QUIC connection. This is a connection-level role and does not imply any restriction on bundle transfer direction.

Qubicle Session:
: The period during which a QUIC connection is established between two Qubicle peers. A session begins when the QUIC handshake completes and ends when the QUIC connection closes. Both client and server are equal peers for the purpose of bundle transfer.

# Applicability Statement {#applicability}

Qubicle adds no transport machinery of its own, so its applicability to a
given environment is that of QUIC itself: Qubicle SHOULD NOT be used where
the QUIC transport cannot be expected to perform well. With the default
transport parameters and timer values of general-purpose QUIC
implementations, this primarily means deployments where round-trip times
(RTTs) remain under a few seconds, so that QUIC's handshake and loss-recovery
behavior are well matched to the path.

This boundary is not fixed. QUIC's transport parameters and timers can be
adjusted to longer-delay paths ({{timer-tuning}}), and a profile of QUIC
developed for a particular environment, whether by tuning, by use of
optional QUIC features such as 0-RTT resumption ({{zero-rtt}}), or by other
means, applies to Qubicle sessions without modification. Where such a
profile exists for an environment, Qubicle's applicability follows it. For
extremely high-delay or disrupted environments such as deep space
communications (e.g., Earth-Mars links with multi-minute RTTs) in the
absence of such a profile, specialized protocols like the Licklider
Transmission Protocol (LTP) {{?RFC5326}} may be more appropriate.

Similarly, the SVCB-based DNS service discovery mechanism ({{dns-example}})
SHOULD NOT be used in environments where DNS itself might not perform well.
DNS-based discovery is NOT RECOMMENDED for use in DTN environments where DNS
infrastructure is unavailable, network disruptions cause failed lookups or
stale cached records, DNSSEC validation fails due to a mismatch between
query RTT and valid signature lifetimes, or DNS query overhead is
significant relative to available bandwidth.
For such environments, implementations SHOULD support alternative CL
provisioning mechanisms including manual configuration with pre-planned
contact schedules, contact graph routing protocols that maintain topology
independently of DNS, or out-of-band metadata distribution through mission
management plane channels.

A hybrid approach is RECOMMENDED for nodes bridging Internet and
deep-space networks: use Qubicle with DNS discovery for Internet-side
connections, and use alternate mission management planes for space-side
connections.

# Protocol Overview

## Connection Establishment

A Qubicle session is established by initiating a QUIC connection to a peer. The QUIC handshake authenticates both peers via TLS 1.3 {{!RFC9001}} as described in {{peer-node-id}}.

The ALPN identifier for Qubicle is `qbcl`.

## Peer Node Identification {#peer-node-id}

Qubicle does not exchange Node IDs in-band. A peer's Node ID is the Node ID
carried in the `BundleEID` OtherName ({{!RFC9174, Section 4.4.2}}) of the
certificate it presents during the TLS handshake. Accordingly, both client
and server MUST present a certificate containing a `BundleEID` OtherName,
and each peer MUST validate the certificate of the other as described in
{{transport-security}}.

The sole exception is a deployment in which a peer has already been
configured with the Node ID of the remote peer by other means (e.g., a
contact plan or manual configuration keyed on the remote address). Such a
peer MAY accept a certificate lacking a `BundleEID` OtherName and associate
the configured Node ID with the session instead. If the peer has a configured
Node ID and the certificate also contains a `BundleEID` OtherName, the two
MUST match; a mismatch MUST be treated as a certificate validation failure
and the handshake terminated.

The means by which a peer obtains the rendezvous information (address and
port) for a remote Node ID, whether by manual configuration, contact plan,
or DNS (see {{dns-example}}), is independent of the identification
mechanism above. Rendezvous information locates a candidate peer; only the
authenticated certificate establishes which Node ID has actually been
reached.

## Reliable Bundle Transfer {#reliable-transfer}

For reliable transfer, bundles are sent as successive items on QUIC
unidirectional streams:

1. The sender selects an existing open sending stream at an item boundary,
   or creates a new unidirectional stream subject to {{flow-control}}.
2. The sender writes a complete bundle wrapped in a definite-length CBOR
   byte string as described in {{stream-content}}.
3. The sender MAY append another bundle after writing the complete preceding
   item, without closing the stream or waiting for acknowledgment of that
   item. Normal QUIC transmission constraints still apply.
4. When finished with the stream, the sender closes it at an item boundary
   by sending a STREAM frame with the FIN bit set.

Receivers MUST support multiple items per stream. Senders remain free to
use a separate stream for each bundle. The receiver recognizes a complete
bundle after consuming its byte-string head and all declared content bytes,
and processes and hands off an accepted bundle to the BPA without waiting
for FIN or the next item. The receiver MUST NOT deliver an incomplete
bundle.

### Transfer Completion {#transfer-completion}

QUIC provides reliable, in-order stream transport. This revision of Qubicle
defines no convergence-layer acknowledgment; per-bundle reception reporting
remains an open issue ({{reception-reporting-todo}}). An individual bundle's
transfer is complete from the sender's transport perspective when all
stream bytes comprising its item, including the CBOR byte-string head,
have been acknowledged by the peer's QUIC endpoint. Acknowledgment of FIN
is not required for per-bundle transport completion.

Implementations MUST associate each outgoing bundle with the stream-offset
range occupied by its item and establish acknowledgment of every byte in
that range before reporting Transmission Success. A successful write to
the QUIC API, available flow-control credit, or acknowledgment of a later
offset alone is not proof that the entire item was acknowledged. QUIC APIs
vary in whether they expose sufficient acknowledgment information; not
every implementation can provide immediate per-bundle completion
indications. An implementation might have to defer a success indication
until its API provides sufficient evidence. This revision does not specify
a portable API mapping or an alternative completion signal for APIs that
cannot provide that evidence.

This completion signal is generated by the peer's QUIC transport
implementation upon receipt of the data. It confirms that the data reached
the peer's QUIC endpoint; it does not confirm that the peer's convergence
layer has read the data, nor that the peer's BPA has accepted the bundle. In
particular, data acknowledged by the QUIC endpoint but not yet consumed by
the receiving application can be lost if the receiving process fails.
Receivers SHOULD therefore read stream data promptly and pass each completed
bundle to the BPA without unnecessary delay, so as to minimize the interval
during which acknowledged data is held only in transport buffers.

Transport completion is consequently distinct from complete reception and
handoff by the remote CLA, and from reception processing by the remote BPA.
It is not a sufficient basis for a sending BPA to conclude that the
next-hop BPA has received a bundle. Where such assurance is required, it
MUST come from an appropriate bundle-layer mechanism, not be inferred from
transport completion. Ordinary status reporting ({{!RFC9171, Section 6.1}})
is not by itself a complete forwarding-node solution: the bundle source
controls the primary-block report request and report-to destination. The
integration needed for forwarding-node assurance remains unspecified in
this revision; see {{reception-reporting-todo}}.

In terms of the convergence layer service model, Qubicle uses the following
per-bundle indications to the BPA, subject to the API limitations above:

Transmission Success:
: Acknowledgment of every byte in the item's stream-offset range, including
  its CBOR head, has been established. Neither stream closure nor
  acknowledgment of subsequent items is required.

Transmission Failure:
: The stream was terminated through cancellation ({{cancellation}}), or the
  connection closed, before transport completion of this item could be
  established. This does not prove that the receiving BPA did not receive
  the bundle.

Reception Success:
: The complete byte-string head and exactly the declared content length
  have been consumed, and the accepted bundle has been passed to the BPA,
  without waiting for FIN or another item. This handoff is distinct from
  the BPA's reception processing.

Reception Failure:
: The item was rejected, truncated at clean stream end, or interrupted by
  stream cancellation or connection loss before completion and handoff;
  any partial bundle data has been discarded.

A later truncated, malformed, or interrupted item does not revoke Reception
Success for bundles already completed and handed off. Likewise, a later
stream reset or connection loss does not revoke an established Transmission
Success for an earlier item. Transport success still does not establish
remote CLA handoff or BPA reception. A sender MUST NOT issue both
Transmission Success and Transmission Failure for the same transmission
attempt. Once either terminal indication has been issued, later transport
events, including delayed acknowledgments after cancellation, do not
reverse it. A retry is a separate transmission attempt.

Qubicle does not provide intermediate progress indications; a BPA requiring
them can observe stream-level flow control state where the QUIC
implementation exposes it.

#### Open Issue: Bundle Reception Reporting {#reception-reporting-todo}

TODO: Specify how a forwarding BPA obtains a positive, bundle-by-bundle
indication that the next-hop BPA has received a bundle, beyond
acknowledgment of its bytes by QUIC. Evaluate incorporation of, or a
reference to, the Bundle Protocol Hop Reception Report Request Block
proposal: <https://github.com/ekline/draft-dtn-hop-reception-report>.
Resolve applicability, correlation, peer authentication, and behavior when
reports are unsupported, declined, delayed, or lost. This revision does not
yet specify that integration.

The candidate mechanism requests an ordinary Bundle Status Report addressed
using the Previous Node block. It complements transport completion and does
not transfer custody. The repository link is informational; the proposal
is not a normative dependency of this revision, which adds neither an
acknowledgment frame nor a requirement for reception reporting.

A missing reception report does not establish failure. Transport success
without a report does not distinguish unsupported reporting from receiver
failure after transport acknowledgment.

### Stream Content {#stream-content}

Each reliable-transfer stream carries a CBOR Sequence {{!RFC8742}} whose
items MUST be definite-length CBOR byte strings. Each byte string MUST
contain the complete encoding of exactly one bundle. Its declared length
is the encoded bundle length in octets, excluding the byte string head
({{!RFC8949, Section 3.1}}). The sender MUST know the outgoing encoded
bundle's length before writing the head.

The sequence has no enclosing array, bundle count, or separator. QUIC
STREAM frame boundaries have no significance for this framing. A sender
MUST NOT interleave bytes of different bundle items within one stream.

Unwrapped bundles, indefinite-length byte strings, and other outer CBOR
types are protocol errors. An empty byte string cannot contain a bundle
and is a protocol error. The first octet of each item is interpreted as
follows:

| First Octet | Interpretation |
|-------------|----------------|
| 0x40 - 0x5b | Definite-length CBOR byte string head; a zero content length is a protocol error |
| any other value | Protocol error |
{: #tab-first-octet align="left" title="Stream Content Dispatch"}

The receiver consumes the byte-string head and exactly the declared number
of content octets to recognize a complete bundle. It processes and hands
off an accepted bundle without waiting for FIN or the next item. Bytes
following a complete item begin the next item.

FIN is valid at an item boundary, including before any item: an empty,
cleanly closed stream is an empty sequence. A clean stream end partway
through a byte-string head or its declared content is malformed framing.
A clean stream end requires FIN and receipt of all bytes preceding the
final stream offset; FIN alone does not establish that missing bytes have
arrived. For malformed framing, the receiver MUST close the connection
with `QBCL_PROTOCOL_ERROR`. Stream reset or connection loss instead
interrupts any incomplete transfer ({{transfer-completion}}); it is not a
clean but malformed stream end. The receiver MUST NOT deliver an incomplete
bundle. Bundles already completed and handed off remain completed even if
a later item is truncated, malformed, or interrupted.

Within each byte string, the first content octet identifies BPv7 by 0x9f
(the indefinite-length outer array, {{!RFC9171, Section 4.1}}), or BPv6 by
its version octet 0x06 ({{?RFC5050}}). Content that does not begin with a
recognized bundle version encoding is a protocol error, and the receiver
MUST close the connection with `QBCL_PROTOCOL_ERROR`. Qubicle
implementations MUST support BPv7 bundles. Support for other bundle
versions, including BPv6, is OPTIONAL; a receiver that recognizes but does
not support the version of an incoming bundle SHOULD cancel the stream
with `QBCL_UNSUPPORTED_BUNDLE_VERSION` rather than treating it as a protocol
error. This cancellation affects the stream, not just that item; see
{{cancellation}}. The wrapper does not change the inner BPv7 encoding,
including its indefinite-length outer array.

Declaring the bundle length before its content allows the receiver to make
admission and storage decisions while transfers compete for stream slots,
flow-control credit, and processing resources. The declaration supports
these decisions but does not guarantee progress or prevent starvation.

Streaming, including cut-through forwarding, remains possible when the
exact outgoing encoded length is known before transmission. Forwarding an
unknown-length outgoing encoding is not supported. The framing requirement
applies only to reliable stream transfers; unreliable transfers retain
{{BTP-U}} encapsulation ({{unreliable-transfer}}).

### Bundle Flow in Both Directions

Both peers can send bundles simultaneously. Each peer creates unidirectional streams to send its bundles. QUIC stream IDs inherently separate client-initiated streams (IDs 2, 6, 10...) from server-initiated streams (IDs 3, 7, 11...), ensuring no collision between the two directions of bundle flow.

Qubicle uses only unidirectional streams. Peers SHOULD set the
`initial_max_streams_bidi` transport parameter to 0. A peer that
nonetheless receives a bidirectional stream MUST treat this as a protocol
error and close the connection with `QBCL_PROTOCOL_ERROR`.

### Stream Selection and Priority

A sender MAY implement bundle priority by scheduling data across multiple
open streams and selecting which pending bundle to append to each stream,
or whether to open a new stream. It MAY also use stream-priority facilities
exposed by its QUIC implementation
({{!RFC9000, Section 2.3}}). Qubicle defines no priority signaling on the
wire. Scheduling remains subject to available stream and connection
flow-control credit and QUIC congestion control. Supplying data to a QUIC
implementation does not guarantee immediate transmission or preemption of
data already queued inside it.

A stream can serve as a persistent scheduling lane for successive bundles;
"lane" is explanatory, not a separate protocol object. Bundles on one
stream are serialized: a subsequent bundle cannot overtake an earlier item,
and bytes of different items MUST NOT be interleaved. Independent
scheduling requires different streams. Queued priority classes are distinct
from these lanes. The number of lanes and their relationship to priority
classes are implementation policy within QUIC's limits ({{flow-control}});
there is no mandatory priority-to-stream mapping.

If every usable stream has lower-priority items ahead of a newly arrived
higher-priority bundle and no new stream can be opened, that bundle can be
delayed. Strict priority can
indefinitely defer lower-priority transfers. If favored streams cannot
progress, refusing to service other eligible streams can leave usable
contact capacity idle. Fairness, admission policy, and any reservation of
capacity for urgent traffic are implementation matters; Qubicle neither
fixes the number of priority classes nor requires fair scheduling.

### Flow Control and Stream Limits {#flow-control}

Qubicle defines no additional stream-count negotiation or parallelism
limit. For each sending direction, stream creation follows the peer's
`initial_max_streams_uni` and subsequent `MAX_STREAMS`, as specified in
{{!RFC9000, Section 4.6}}. These are cumulative stream-opening limits, not
merely limits on the number currently open. Closing a stream does not by
itself grant permission to open another; the receiver supplies additional
credit. The two peers' advertisements govern opposite sending directions.
Implementations may choose to use fewer streams.

Permission to open a stream is distinct from the ability to transmit data.
Active streams remain subject to stream-level and connection-level
flow control ({{!RFC9000, Section 4.1}}), congestion control, and sender
scheduling. QUIC's `initial_max_stream_data_uni` and `MAX_STREAM_DATA` limit
cumulative stream
data, while `initial_max_data` and `MAX_DATA` limit cumulative data across
streams. These are not fixed limits on data in flight or total bundle
storage. A receiver that is temporarily unable to accept more data can
withhold additional credit, causing the sender to pause once available
credit is exhausted, without error. Receivers SHOULD grant credit based on
the resources actually available for admitted transfers; see
{{flow-control-sizing}}.

Persistent streams reduce dependence on replenishment of stream-opening
credit, but still require replenishment of stream-level and connection-level
byte credit. If further streams cannot be opened, a sender can continue
appending items to existing sending streams subject to these constraints.

## Unreliable Bundle Transfer {#unreliable-transfer}

For unreliable transfer, bundles are sent using QUIC datagrams {{!RFC9221}} with {{BTP-U}} framing.

Support for the unreliable service is OPTIONAL. An implementation offering
it MUST advertise the `max_datagram_frame_size` transport parameter. The
unreliable service is available on a session only if both peers have
advertised this parameter; otherwise neither peer sends DATAGRAM frames and
only the reliable service is available. Implementations SHOULD make the
availability of the unreliable service known to the BPA so that it can
select an appropriate service for each bundle.

Each QUIC DATAGRAM frame carries one or more {{BTP-U}} messages, and plays
the role of the Link-layer PDU in the {{BTP-U}} model: it is delivered in
its entirety or not at all. A sender MUST NOT construct a datagram larger
than the peer's advertised `max_datagram_frame_size`, and SHOULD size
datagrams to fit within the current path MTU so that they are not dropped by
the QUIC layer. {{BTP-U}} segmentation, reassembly, transfer identification,
and optional repetition apply unchanged.

The {{BTP-U}} Transfer Window size ({{BTP-U, Section 5}}) MUST be configured
consistently at both peers; how this is done is out of scope for this
document.

QUIC DATAGRAM frames are subject to QUIC congestion control
({{!RFC9221, Section 5}}), satisfying the {{BTP-U}} requirement that it not
be deployed without congestion control where congestion may occur.

The mapping of bundle priority to {{BTP-U}} transfer interleaving is an implementation matter.

## Connection Termination

A session ends when the QUIC connection closes, whether by an explicit
`CONNECTION_CLOSE` from either peer or by expiry of the idle timeout
({{keepalive}}). Idle timeout is a normal way for a session to end, not an
error; a peer with further bundles to send simply establishes a new session.

A peer wishing to end a session gracefully SHOULD stop opening new streams,
starting new bundle items on existing streams, and initiating new {{BTP-U}}
transfers. It SHOULD finish current items where feasible, close its sending
streams at item boundaries, and allow outstanding reliable transfers in
both directions to complete before closing the connection with
`QBCL_NO_ERROR`. This revision defines no session-level shutdown signal to
prevent the remote peer from starting further items or streams, so a peer
SHOULD bound this wait with a local timer.

A peer MAY instead close the connection immediately at any time. If it does
so deliberately while transfers are in progress it SHOULD use
`QBCL_SHUTTING_DOWN`; other codes in {{error-codes}} apply for error
conditions.

When a connection closes for any reason, the receiver MUST discard any
partially received bundle data, as in {{cancellation}}. Bundles already
completed and handed off remain completed. The sender reports Transmission
Failure for outstanding items whose transport completion could not be
established, subject to the acknowledgment/API limitations in
{{transfer-completion}}; established Transmission Success indications are
not revoked. Re-forwarding is subject to BPA policy, since a failure to
establish transport completion does not prove non-reception by the remote
BPA. Any incomplete {{BTP-U}} transfers on the session are likewise lost.

Two peers may establish connections to each other concurrently, resulting in
more than one session between the same pair of Node IDs. This is permitted
and is not an error. A peer MAY choose to gracefully close a redundant
session, but MUST accept bundles received on any established session.

## Transfer Cancellation {#cancellation}

Either peer may cancel a reliable-transfer stream. `RESET_STREAM` and
`STOP_SENDING` affect the stream, not one selected bundle item. This
revision defines no per-bundle cancellation protocol.

A sender cancels a stream by sending `RESET_STREAM` with an appropriate
error code ({{error-codes}}), subject to QUIC's stream-state rules
({{!RFC9000, Section 3}}). A sender might do this when a bundle's lifetime
expires or the BPA withdraws a bundle whose item has already been started.
A sender MUST NOT truncate an item and then append another item on the same
stream: that would corrupt framing. It must finish the item or terminate
the stream.

A receiver cancels a stream by sending `STOP_SENDING` with an appropriate
error code, for example when a bundle exceeds its storage or policy limits,
or when its BPA is shutting down. The sender MUST handle `STOP_SENDING`
according to {{!RFC9000, Section 3.5}} and the stream-state rules in
{{!RFC9000, Section 3.3}}. In particular, a stream in the Send state still
requires `RESET_STREAM` even if all item bytes written so far have been
acknowledged; per-bundle transport completion does not close the stream.

Cancelling a stream can abandon its current partial bundle and later
queued or outstanding bundles. On receiving `RESET_STREAM`, a receiver
MUST discard any partially received bundle data for that stream and MUST
NOT deliver a partial bundle to the BPA. A receiver that has sent
`STOP_SENDING` MUST likewise discard partial data and any subsequently
received data on that stream. Cancellation does not revoke bundles already
handed to the receiving BPA.

An application error code describes a reason for stream cancellation; it
does not identify the item that caused it. The sender may have queued
multiple items ahead of the receiver. Cancellation can also race with
transport acknowledgment and is not a reliable per-bundle rejection
notification. The sender reports failure for affected items whose transport
completion has not been established, as described in
{{transfer-completion}}, without revoking established transport success.
Bundles MAY be re-forwarded subject to BPA policy; a failure indication
does not establish that they were not received.

A receiver enforcing a maximum receivable bundle size can reject a bundle
based on its declared length as soon as it has read the byte string head
({{stream-content}}), cancelling the stream with `QBCL_BUNDLE_TOO_LARGE`.
Payload bytes, including later items, may already have been sent or
received at that point. A receiver whose available storage cannot
accommodate a transfer can cancel the stream with `QBCL_STORAGE_EXHAUSTED`.
Admission and credit policies SHOULD consider the declared sizes of active
bundles and allow admitted transfers to progress toward completion, rather
than leaving many partial transfers
unable to complete ({{flow-control-sizing}}).

If a receiver elects to skip a rejected item and continue parsing instead
of cancelling the stream, it MUST consume exactly the declared content
length before reading the next head. This does not save transmission
bandwidth or notify the sender of rejection, and does not relax the
connection-close requirement for malformed framing.

The initial stream flow-control allowance is not an advertised maximum
bundle size. Receivers need sufficient available stream and connection
credit to read each head, which can occupy up to nine octets and itself
consumes credit. They can consume or spool data and replenish credit
incrementally; they need not grant credit for an entire bundle at once.

Cancellation of unreliable transfers is governed by {{BTP-U}}.

## Keepalive {#keepalive}

Qubicle relies on QUIC's native idle timeout mechanism. Peers negotiate the `max_idle_timeout` transport parameter during connection establishment. See {{timer-tuning}} for guidance on selecting this and related values.

Where a session must be kept alive across periods with no bundles to send,
implementations MAY use QUIC PING frames ({{!RFC9000, Section 10.1.2}})
where the QUIC implementation exposes such a facility. Keepalives should be
balanced against the idle timeout guidance in {{timer-tuning}}: a session
that is deliberately allowed to time out and re-established later can be
preferable to keeping one alive through a scheduled link outage.

# Error Codes {#error-codes}

Qubicle defines a single space of application error codes, used in the QUIC
`CONNECTION_CLOSE` frame (with the application-level frame type, see
{{!RFC9000, Section 19.19}}), `RESET_STREAM` frame, and `STOP_SENDING`
frame. QUIC application error codes are 62-bit unsigned integers.

| Code | Name | Description |
|------|------|-------------|
| 0x00 | QBCL_NO_ERROR | Graceful connection closure, no error |
| 0x01 | QBCL_PROTOCOL_ERROR | Peer violated this specification |
| 0x02 | QBCL_TRANSFER_CANCELLED | Transfer aborted for a reason not otherwise specified (e.g., bundle lifetime expired, BPA withdrew the bundle) |
| 0x03 | QBCL_BUNDLE_TOO_LARGE | Incoming bundle exceeds the receiver's maximum receivable bundle size |
| 0x04 | QBCL_STORAGE_EXHAUSTED | Receiver temporarily cannot accept bundles; the sender MAY retry later |
| 0x05 | QBCL_SHUTTING_DOWN | Peer is deliberately closing the session; no fault is implied |
| 0x06 | Reserved | |
| 0x07 | QBCL_UNSUPPORTED_BUNDLE_VERSION | Receiver does not support the version of the bundle being transferred |
{: #tab-error-codes align="left" title="Qubicle Error Codes"}

`QBCL_NO_ERROR` is only meaningful on `CONNECTION_CLOSE`. A cancelled
transfer always has a cause, so peers MUST NOT use `QBCL_NO_ERROR` in
`RESET_STREAM` or `STOP_SENDING`; a peer receiving it there SHOULD treat it
as `QBCL_TRANSFER_CANCELLED`.

When a peer closes the session deliberately while transfers are still in
progress, it SHOULD use `QBCL_SHUTTING_DOWN` on `CONNECTION_CLOSE` rather
than `QBCL_NO_ERROR`, so that the remote peer can distinguish an orderly
shutdown with collateral transfer failures from an idle close.

A peer receiving an unrecognized error code MUST treat it as
`QBCL_PROTOCOL_ERROR` on `CONNECTION_CLOSE` and as `QBCL_TRANSFER_CANCELLED`
on `RESET_STREAM` or `STOP_SENDING`.

# Security Considerations

## Transport Security {#transport-security}

QUIC mandates TLS 1.3 for all connections, providing confidentiality, integrity, and authentication. Qubicle inherits these security properties.

As specified in {{peer-node-id}}, Qubicle requires mutual certificate
authentication: both client and server present certificates, and each peer
validates the other's certificate before exchanging bundles. Certificate
validation, including the `BundleEID` OtherName checks and the applicable
certificate profile, SHOULD follow {{!RFC9174, Section 4.4}}. Automated
certificate provisioning is available via the ACME extensions defined in
{{?RFC9891}}.

## Bundle Security

Transport security protects bundles in transit between adjacent nodes. For end-to-end bundle security, implementations SHOULD use BPSec {{!RFC9172}}.

## Denial of Service

QUIC provides built-in protection against many denial-of-service attacks, including address validation and amplification prevention.

QUIC flow control ({{flow-control}}) is the primary means of bounding the
resources a peer can consume; receivers SHOULD configure it accordingly
rather than accepting unbounded data and discarding it afterwards.

## 0-RTT Considerations {#zero-rtt}

QUIC 0-RTT data is not protected against replay. Because a replayed bundle
transfer could cause a bundle to be received and forwarded more than once,
implementations SHOULD NOT send bundles as 0-RTT data, and servers MAY
decline to accept 0-RTT altogether.

0-RTT resumption could nonetheless materially reduce session establishment
cost on long-delay paths between peers that reconnect frequently. The
consequence of a replayed transfer is duplicate receipt of a bundle, which a
BPA can detect from the combination of source node ID and creation timestamp
({{!RFC9171, Section 4.2.7}}) together with any fragment offset, except for
anonymous bundles, and whose impact is bounded where BPSec ({{!RFC9172}})
protects the bundle. Whether this risk is acceptable in a given environment
is a matter for operational experience and for any QUIC profile applicable
to that environment ({{applicability}}); a deployment that does permit 0-RTT
SHOULD ensure that its BPAs perform duplicate bundle detection.

# Operational Considerations

## Version Negotiation

To resist ossification, Qubicle endpoints are RECOMMENDED to support QUIC
version 2 {{!RFC9369}} and compatible version negotiation {{!RFC9368}}.
Qubicle operates identically over any QUIC version providing the features
used in this document.

## Timer and Transport Parameter Tuning {#timer-tuning}

QUIC connection behavior is governed by a number of timers. Some are
negotiated via transport parameters during the handshake (e.g.,
`max_idle_timeout` and `max_ack_delay`), while others are derived locally
from measured path characteristics (e.g., the Probe Timeout (PTO) computed
from RTT estimates per {{!RFC9002}}, seeded by an implementation's initial
RTT value).

In many deployments, particularly those on the terrestrial Internet, the
default values recommended by {{!RFC9000}} and {{!RFC9002}} and used by
general-purpose QUIC implementations are expected to be adequate, and
Qubicle implementations MAY use them unchanged.

However, Qubicle peers may possess knowledge of end-to-end path
characteristics that is unavailable to the QUIC transport itself, for
example from contact plans, orbital mechanics, or link scheduling
information in non-terrestrial deployments. Such information may indicate
expected propagation delays, predictable link outages, or highly asymmetric
paths for which Internet defaults are inappropriate (e.g., an idle timeout
short enough to be tripped by a scheduled link gap, or an initial RTT small
enough to cause spurious PTO retransmissions during the handshake).
Implementations SHOULD allow these timers and transport parameters to be
configured on a per-peer basis so that they can be adjusted using such
knowledge. The means by which such path knowledge is obtained, and the
specific adjustments derived from it, are out of scope for this document and
may be the subject of environment-specific QUIC profiles.

## Flow Control Sizing {#flow-control-sizing}

Stream-opening credit, per-stream data credit, and connection-level data
credit ({{flow-control}}) interact with receiver admission and sender
scheduling. Partial transfers can collectively occupy storage, stream
slots, and connection credit without any bundle completing. This can stall
useful progress or waste a finite contact. Receiver admission and credit
policies SHOULD consider the declared sizes of active bundles and allow
admitted transfers to progress toward completion. The length prefix
supplies information for that policy; it is not a deadlock-prevention
mechanism and does not prevent starvation.

The receiver needs sufficient initial stream and connection credit to read
the first CBOR byte-string head, which can occupy up to nine octets, and
sufficient replenished credit for subsequent heads. Each head itself
consumes both kinds of flow-control credit. A receiver need not grant
credit for an entire bundle at once: it can consume or spool data and
replenish credit incrementally. Granting credit SHOULD reflect actual
available resources, including storage retained for partial bundles even
after data has been consumed from transport buffers. Declared lengths are
untrusted input and MUST NOT cause unchecked allocation. The initial
flow-control allowance is not an advertised maximum bundle size; size and
resource-based rejection are described in {{cancellation}}. On a persistent
stream, completing an item neither closes the stream nor replenishes byte
credit automatically. Successive items reduce the need for new
stream-opening credit but still depend on replenishment of byte credit.
Receiver admission policies must also account for later items that the
sender may already have queued on the same stream.

Throughput depends on available credit and how promptly it is replenished
relative to the path RTT, not on treating QUIC's cumulative limits as fixed
receive windows. Too little available credit, or replenishment that arrives
too late, can leave a high-bandwidth, high-delay path underutilized. The
appropriate balance between storage commitment, parallelism, and
throughput is deployment-specific, and the same out-of-scope path knowledge
discussed in {{timer-tuning}} can inform it.

## Convergence Layer Fallback

As noted in {{RFC9308}}, some networks block UDP traffic such that
Qubicle connections cannot be established. Bundle Protocol Agents that
employ Qubicle are RECOMMENDED to support additional Convergence Layers,
e.g. TCPCLv4 {{!RFC9174}}.

## Coexistence With Other UDP-based Convergence Layers

It is RECOMMENDED that Qubicle implementations use a dedicated UDP port for
operational simplicity.

Bundle Protocol Agents that employ Qubicle and other UDP-based Convergence
Layers on the same UDP port MUST be able to disambiguate received datagrams
in order to route them to the correct CLA. For UDP CLs that use DTLS,
{{!RFC9443}} provides the required guidance to disambiguate QUIC traffic
from DTLS-encapsulated CL traffic.

## Finding a Qubicle Endpoint Via DNS {#dns-example}

Qubicle senders may be manually provisioned with a hostname
(or IP addresses) and UDP port corresponding to the listening Qubicle
endpoint for a peer Bundle Protocol Agent.
If only a hostname is known but a port is not, {{!RFC9460}} SVCB
Resource Records may be looked up to find a listening
UDP port and confirm expected ALPN configuration.

Consider this zone file for `example.`:

~~~
;; zone: example.
;
$ORIGIN example.
_dtn-bundle._tcp.mars-orbiter IN SRV 10 20 4556 cloud-agent.example.
_qbcl.mars-orbiter IN SVCB 0 cloud-agent.example.

cloud-agent IN A    192.0.2.1
cloud-agent IN AAAA 2001:db8::1
cloud-agent IN SVCB 10 . (
    ipv4hint=192.0.2.1
    ipv6hint=2001:db8::1
    port=1234 alpn="qbcl")
~~~
{: artwork-name="dns-zone-example"}

A BPA supporting both TCPCLv4 {{!RFC9174}} and Qubicle may attempt to
resolve an SRV record for the `_dtn-bundle._tcp` prefixed hostname. A BPA
that supports Qubicle
might also issue DNS SVCB queries for the {{AttrLeaf}} prefix "_qbcl". The
sample above indicates that `mars-orbiter.example.` has an SVCB record in
`AliasMode` referring to `cloud-agent.example.`  The SVCB record associated
with `cloud-agent.example.` contains all required QUIC transport rendezvous
information.

# IANA Considerations

## ALPN Identifier

IANA is requested to register the following ALPN identifier in the "TLS Application-Layer Protocol Negotiation (ALPN) Protocol IDs" registry:

| Protocol | Identification Sequence | Reference |
|----------|------------------------|-----------|
| Qubicle | 0x71 0x62 0x63 0x6C ("qbcl") | This document |
{: #tab-alpn align="left" title="ALPN Registration"}

## AttrLeaf Node Name

Per {{AttrLeaf}}, IANA is requested to add the following entry to the DNS
"Underscored and Globally Scoped DNS Node Names" registry:

| RR Type | _NODE NAME | Reference     |
|---------|------------|---------------|
| SVCB    | _qbcl      | this document |
{: #tab-attrleaf align="left" title="AttrLeaf Registration"}

## Application Error Codes

IANA is requested to create a new registry "Qubicle Error Codes".
Values in this registry are 62-bit unsigned integers, matching the range of
QUIC application error codes. Each entry consists of a Code, a Name, a brief
Description, and a Reference.

The registration policy for this registry is Specification Required
{{!RFC8126}} for codes in the range 0x00 to 0x3FFFFFFF. Codes in the
range 0x40000000 to 0x3FFFFFFFFFFFFFFF are reserved for Private Use
{{!RFC8126}} and are not assigned by IANA.

The initial contents of the registry are:

| Code | Name | Description | Reference |
|------|------|-------------|-----------|
| 0x00 | QBCL_NO_ERROR | Graceful connection closure, no error | This document |
| 0x01 | QBCL_PROTOCOL_ERROR | Peer violated this specification | This document |
| 0x02 | QBCL_TRANSFER_CANCELLED | Transfer aborted, unspecified reason | This document |
| 0x03 | QBCL_BUNDLE_TOO_LARGE | Bundle exceeds receiver's maximum size | This document |
| 0x04 | QBCL_STORAGE_EXHAUSTED | Receiver temporarily cannot accept bundles | This document |
| 0x05 | QBCL_SHUTTING_DOWN | Peer is deliberately closing the session | This document |
| 0x06 | Reserved | | This document |
| 0x07 | QBCL_UNSUPPORTED_BUNDLE_VERSION | Receiver does not support the bundle version | This document |
| 0x08-0x3FFFFFFF | Unassigned | | |
| 0x40000000-0x3FFFFFFFFFFFFFFF | Reserved for Private Use | | This document |
{: #tab-error-registry align="left" title="Error Code Registry"}

--- back

# Acknowledgments
{: numbered="false"}

The authors thank Carlo Caini for discussions of QUIC stream concurrency,
sender scheduling, and flow control that helped clarify the prioritization
and framing requirements in this document.
