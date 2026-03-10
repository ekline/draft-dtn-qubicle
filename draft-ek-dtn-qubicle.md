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
  email: <rtaylor@aalyria.com>
- fullname: Erik Kline
  organization: Aalyria Technologies
  email: <ek.ietf@gmail.com>

normative:
  AttrLeaf: RFC8552
  BTP-U: I-D.ietf-dtn-btpu

informative:
  RFC9308:

--- abstract

This document specifies a minimal convergence layer protocol for transferring Bundle Protocol version 7 (BPv7) bundles over QUIC. The protocol leverages QUIC's native capabilities for reliable streaming, connection management, and security, requiring no application-layer framing for reliable transfers. Unreliable transfers use the Bundle Transfer Protocol - Unidirectional (BTP-U) over QUIC datagrams.

--- middle

<!--

XXX

To be considered:

* Port Selection and ALPN - Protocol identification and negotiation
* Connection Migration - Handling mobile nodes and NAT rebinding
* Connection Termination - Graceful shutdown and timeout handling

* receiver shutdown on 2nd bundle
* Error codes???

-->

# Introduction

Bundle Protocol version 7 (BPv7) {{!RFC9171}} requires Convergence Layer
Adapters (CLAs) to transfer bundles between nodes. This document specifies
the QUIC Bundle Protocol Convergence Layer (QBCL or "qubicle"),
a minimal CLA using QUIC {{!RFC9000}} that embraces QUIC's native
capabilities rather than layering additional protocol machinery.

The design philosophy is simple: QUIC already provides reliable streams, multiplexing, flow control, congestion control, and integrated security. This specification adds only what is strictly necessary to transfer bundles.

The protocol provides two services:

Reliable Service:
: Bundles are transferred on QUIC streams with guaranteed delivery.

Unreliable Service:
: Bundles are transferred via QUIC datagrams {{!RFC9221}} using {{BTP-U}} framing.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

Client:
: The Qubicle peer that initiates the QUIC connection. This is a connection-level role and does not imply any restriction on bundle transfer direction.

Server:
: The Qubicle peer that accepts the QUIC connection. This is a connection-level role and does not imply any restriction on bundle transfer direction.

Qubicle Session:
: The period during which a QUIC connection is established between two Qubicle peers. A session begins when the QUIC handshake completes and ends when the QUIC connection closes. Both client and server are equal peers for the purpose of bundle transfer.

# Protocol Overview

## Connection Establishment

A Qubicle session is established by initiating a QUIC connection to a peer. The QUIC handshake provides mutual authentication via TLS 1.3 {{!RFC9001}}.

The ALPN identifier for Qubicle is `qbcl`.

## Reliable Bundle Transfer {#reliable-transfer}

For reliable transfer, each bundle is sent on a dedicated QUIC unidirectional stream:

1. The sender creates a new unidirectional stream.
2. The sender writes the complete bundle (CBOR-encoded per {{!RFC9171}}) to the stream.
3. The sender closes the stream by sending a STREAM frame with the FIN bit set.

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

For unreliable transfer, bundles are sent using QUIC datagrams {{!RFC9221}} with {{BTP-U}} framing.

Each QUIC datagram contains one or more {{BTP-U}} messages. The {{BTP-U}} specification defines segmentation, reassembly, transfer identification, and optional repetition for probabilistic reliability.

Implementations MUST negotiate the QUIC `max_datagram_frame_size` transport parameter to enable datagram support.

The mapping of bundle priority to {{BTP-U}} transfer interleaving is an implementation matter.

## Connection Termination

To terminate a session, a peer closes the QUIC connection using CONNECTION_CLOSE. Application-specific error codes are defined in {{error-codes}}.

A peer MAY close the connection at any time. In-flight reliable transfers on incomplete streams will fail; the BPA is notified of the failure.

## Keepalive

Qubicle relies on QUIC's native idle timeout mechanism. Peers negotiate the `max_idle_timeout` transport parameter during connection establishment.

If application-layer liveness detection is required, implementations MAY send QUIC PING frames.

# Error Codes {#error-codes}

The following application error codes are defined for use with QUIC CONNECTION_CLOSE:

| Code | Name | Description |
|------|------|-------------|
| 0x00 | NO_ERROR | Graceful closure, no error |
{: #tab-error-codes align="left" title="Qubicle Error Codes"}

# Security Considerations

## Transport Security

QUIC mandates TLS 1.3 for all connections, providing confidentiality, integrity, and authentication. Qubicle inherits these security properties.

<!-- XXX -->
Implementations SHOULD require peer certificate authentication. The Node ID in the transport parameter SHOULD match an identity in the peer's certificate. The `BundleEID` OtherName form defined in {{?RFC9174, Section 4.4.2}} provides a standard mechanism for embedding DTN Node IDs in X.509 certificates. Automated certificate provisioning is available via the ACME extensions defined in {{?RFC9891}}.

## Bundle Security

Transport security protects bundles in transit between adjacent nodes. For end-to-end bundle security, implementations SHOULD use BPSec {{!RFC9172}}.

## Denial of Service

QUIC provides built-in protection against many denial-of-service attacks, including address validation and amplification prevention.

Implementations SHOULD apply rate limiting on bundle reception to prevent resource exhaustion.

## 0-RTT Considerations

QUIC 0-RTT data is subject to replay attacks. Implementations that enable 0-RTT SHOULD only send bundles that are safe to replay (e.g., bundles with replay protection at the bundle layer).

# Operational Considerations

## Version Negotiation

Qubicle endpoints wishing to combat various ossification vectors are
RECOMMENDED to support version negotiation and the same Bundle transfer
operations described in this memo over QUIC v2 {{!RFC9369}}.

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

## Finding a Qubicle Endpoint Via DNS

Qubicle senders may be manually provisioned with a hostname
(or IP addresses) and UDP port corresponding to the listening Qubicle
endpoint for a peer Bundle Protocol Agent.
If only a hostname is known but a port is not, {{!RFC9460}} SVCB
Resource Records may be looked up to find a listening
UDP port and confirm expected ALPN configuration.

Consider this zone file for `example.`:

~~~
// zone: example.
//
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

A BPA supporting both {{!RFC9174}} may attempt to resolve an SRV record
for the `_dtn-bundle._tcp` prefixed hostname. A BPA that support Qubicle
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

Per {{AttrLeaf}}, IANA is request to add the following entry to the DNS
"Underscored and Globally Scoped DNS Node Names" registry:

| RR Type | _NODE NAME | Reference     |
|---------|------------|---------------|
| SVCB    | _qbcl      | this document |
{: #tab-attrleaf align="left" title="AttrLeaf Registration"}

## Application Error Codes

IANA is requested to create a new registry "Qubicle Error Codes" with the following initial values:

| Code | Name | Reference |
|------|------|-----------|
| 0x00 | NO_ERROR | This document |
| 0x01-0xEF | Unassigned | |
| 0xF0-0xFF | Reserved for Private Use | This document |
{: #tab-error-registry align="left" title="Error Code Registry"}

--- back
