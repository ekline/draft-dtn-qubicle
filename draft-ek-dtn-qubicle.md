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
- fullname: Erik Kline
  organization: Aalyria Technologies
  email: <ek.ietf@gmail.com>
- fullname: Rick Taylor
  organization: Aalyria Technologies
  email: <rtaylor@aalyria.com>

--- abstract

This document specifies a minimal convergence layer protocol for transferring Bundle Protocol version 7 (BPv7) bundles over QUIC. The protocol leverages QUIC's native capabilities for reliable streaming, connection management, and security, requiring no application-layer framing for reliable transfers. Unreliable transfers use the Bundle Transfer Protocol - Unidirectional (BTPU) over QUIC datagrams.

--- middle

# Introduction

Bundle Protocol version 7 (BPv7) {{!RFC9171}} requires Convergence Layer Adapters (CLAs) to transfer bundles between nodes. This document specifies a minimal CLA using QUIC {{!RFC9000}} that embraces QUIC's native capabilities rather than layering additional protocol machinery.

The design philosophy is simplicity: QUIC already provides reliable streams, multiplexing, flow control, congestion control, and integrated security. This specification adds only what is strictly necessary to transfer bundles.

The protocol provides two services:

Reliable Service:
: Bundles are transferred on QUIC streams with guaranteed delivery.

Unreliable Service:
: Bundles are transferred via QUIC datagrams {{!RFC9221}} using BTPU {{!I-D.ietf-dtn-btpu}} framing.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

Client:
: The Qubicle peer that initiates the QUIC connection. This is a connection-level role and does not imply any restriction on bundle transfer direction.

Server:
: The Qubicle peer that accepts the QUIC connection. This is a connection-level role and does not imply any restriction on bundle transfer direction.

Node ID:
: A URI that uniquely identifies a Bundle Protocol node, as defined in {{Section 4.2.5.2 of !RFC9171}}. For the `ipn` scheme, a Node ID has service number 0 (e.g., `ipn:1.0`). For the `dtn` scheme, a Node ID has an empty demux component (e.g., `dtn://node.example/`). See {{node-id-parameter}} for the text representation used by this protocol.

Qubicle Session:
: The period during which a QUIC connection is established between two Qubicle peers. A session begins when the QUIC handshake completes and ends when the QUIC connection closes. Both client and server are equal peers for the purpose of bundle transfer.

# Protocol Overview

## Connection Establishment

A Qubicle session is established by initiating a QUIC connection to a peer. The QUIC handshake provides mutual authentication via TLS 1.3 {{!RFC9001}}.

The ALPN identifier for Qubicle is `qbcl/1`.

During connection establishment, peers exchange Node IDs using QUIC Transport Parameters as specified in {{node-id-parameter}}.

## Reliable Bundle Transfer {#reliable-transfer}

For reliable transfer, each bundle is sent on a dedicated QUIC unidirectional stream:

1. The sender creates a new unidirectional stream.
2. The sender writes the complete bundle (CBOR-encoded per {{!RFC9171}}) to the stream.
3. The sender closes the stream by sending FIN.

The bundle is implicitly framed by the stream boundaries. No length prefix or application-layer framing is required.

The receiver reads data from the stream until FIN is received, then delivers the complete bundle to the BPA.

QUIC guarantees reliable, in-order delivery of stream data. No application-layer acknowledgment is required; the sender can consider the transfer complete when QUIC confirms the stream data has been acknowledged by the peer.

### Bidirectional Bundle Flow

Both peers can send bundles simultaneously. Each peer creates unidirectional streams to send its bundles. QUIC stream IDs inherently separate client-initiated streams (IDs 2, 6, 10...) from server-initiated streams (IDs 3, 7, 11...), ensuring no collision between the two directions of bundle flow.

### Stream Selection and Priority

Senders MAY use QUIC stream priorities to expedite higher-priority bundles. The mapping of bundle priority to QUIC stream priority is an implementation matter.

### Stream Exhaustion

QUIC stream identifiers are 62-bit values, providing an effectively unlimited number of streams per connection. The MAX_STREAMS transport parameter limits concurrent streams, not the total number of streams over a connection's lifetime.

If an implementation reaches practical limits on stream creation, it SHOULD close the connection and establish a new one.

## Unreliable Bundle Transfer {#unreliable-transfer}

For unreliable transfer, bundles are sent using QUIC datagrams {{!RFC9221}} with BTPU {{!I-D.ietf-dtn-btpu}} framing.

Each QUIC datagram contains one or more BTPU messages. The BTPU specification defines segmentation, reassembly, transfer identification, and optional repetition for probabilistic reliability.

Implementations MUST negotiate the QUIC `max_datagram_frame_size` transport parameter to enable datagram support.

The mapping of bundle priority to BTPU transfer interleaving is an implementation matter.

## Connection Termination

To terminate a session, a peer closes the QUIC connection using CONNECTION_CLOSE. Application-specific error codes are defined in {{error-codes}}.

A peer MAY close the connection at any time. In-flight reliable transfers on incomplete streams will fail; the BPA is notified of the failure.

## Keepalive

Qubicle relies on QUIC's native idle timeout mechanism. Peers negotiate the `max_idle_timeout` transport parameter during connection establishment.

If application-layer liveness detection is required, implementations MAY send QUIC PING frames.

# Node ID Transport Parameter {#node-id-parameter}

Peers exchange Node IDs during QUIC connection establishment using a new transport parameter.

The `node_id` transport parameter (see {{iana-transport-param}}) contains the UTF-8 encoded text representation of the sending peer's Node ID. The text representation depends on the URI scheme:

`dtn` scheme:
: Text syntax per {{Section 4.2.5.1.1 of !RFC9171}}: `dtn://<node-name>/` where `<node-name>` identifies the node (e.g., `dtn://example.dtn/`).

`ipn` scheme:
: Text syntax per {{!RFC9758, Section 4}}: `ipn:[<allocator>.]<node>.0` where the components are non-negative integers and the service number is 0 (e.g., `ipn:1.0` or `ipn:977000.1.0`).

Implementations SHOULD support both the `dtn` and `ipn` URI schemes. Other schemes registered in the "Bundle Protocol URI Scheme Types" registry MAY be supported.

The parameter value is encoded as a UTF-8 string without a terminating null character. For example, the Node ID `ipn:1.0` is encoded as the seven octets `0x69 0x70 0x6E 0x3A 0x31 0x2E 0x30` ("ipn:1.0").

A peer MAY omit the `node_id` parameter to avoid exposing its identity on an untrusted network. If the parameter is omitted or contains a value that cannot be parsed as a Node ID for a supported URI scheme, the peer's Node ID is considered unknown.

# Error Codes {#error-codes}

The following application error codes are defined for use with QUIC CONNECTION_CLOSE:

| Code | Name | Description |
|------|------|-------------|
| 0x00 | NO_ERROR | Graceful closure, no error |
{: #tab-error-codes align="left" title="Qubicle Error Codes"}

# Security Considerations

## Transport Security

QUIC mandates TLS 1.3 for all connections, providing confidentiality, integrity, and authentication. Qubicle inherits these security properties.

Implementations SHOULD require peer certificate authentication. The Node ID in the transport parameter SHOULD match an identity in the peer's certificate. The `BundleEID` OtherName form defined in {{?RFC9174, Section 4.4.2}} provides a standard mechanism for embedding DTN Node IDs in X.509 certificates. Automated certificate provisioning is available via the ACME extensions defined in {{?RFC9891}}.

## Bundle Security

Transport security protects bundles in transit between adjacent nodes. For end-to-end bundle security, implementations SHOULD use BPSec {{!RFC9172}}.

## Denial of Service

QUIC provides built-in protection against many denial-of-service attacks, including address validation and amplification prevention.

Implementations SHOULD apply rate limiting on bundle reception to prevent resource exhaustion.

## 0-RTT Considerations

QUIC 0-RTT data is subject to replay attacks. Implementations that enable 0-RTT SHOULD only send bundles that are safe to replay (e.g., bundles with replay protection at the bundle layer).

# IANA Considerations

## ALPN Identifier

IANA is requested to register the following ALPN identifier in the "TLS Application-Layer Protocol Negotiation (ALPN) Protocol IDs" registry:

| Protocol | Identification Sequence | Reference |
|----------|------------------------|-----------|
| Qubicle | 0x71 0x62 0x63 0x6C 0x2F 0x31 ("qbcl/1") | This document |
{: #tab-alpn align="left" title="ALPN Registration"}

## QUIC Transport Parameter {#iana-transport-param}

IANA is requested to register the following transport parameter in the "QUIC Transport Parameters" registry:

| Value | Parameter Name | Reference |
|-------|----------------|-----------|
| TBD | node_id | This document |
{: #tab-transport-param align="left" title="Transport Parameter Registration"}

## Application Error Codes

IANA is requested to create a new registry "Qubicle Error Codes" with the following initial values:

| Code | Name | Reference |
|------|------|-----------|
| 0x00 | NO_ERROR | This document |
| 0x01-0xEF | Unassigned | |
| 0xF0-0xFF | Reserved for Private Use | This document |
{: #tab-error-registry align="left" title="Error Code Registry"}

--- back
