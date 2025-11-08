---
title: "DTN QUIC Bundle Protocol Convergence Layer (qubicle)"
abbrev: "DTN QUIC CL"
category: std

docname: draft-ek-dtn-qubicle-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Internet"
workgroup: "Delay/Disruption Tolerant Networking"
keyword:
 - next generation
 - unicorn
 - sparkling distributed ledger
venue:
  group: "Delay/Disruption Tolerant Networking"
  type: "Working Group"
  mail: "dtn@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/dtn/"
  github: "ekline/draft-dtn-qubicle"
  latest: "https://ekline.github.io/draft-dtn-qubicle/draft-ek-dtn-qubicle.html"

author:
 -
    fullname: Erik Kline
    organization: Aalyria Technologies, Inc.
    email: ek.ietf@gmail.com

normative:
  BPv7: RFC9171
  BTP-U: I-D.ietf-dtn-btpu

informative:
  RFC9308:

...

--- abstract

TODO Abstract


--- middle

<!--

* one new stream per bundle
* QUIC considerations
  * what to do about 0-RTT and replay attacks
    * only ok if you can de-dup Bundles in some way?
    * only for non-Bundle, {0,1} config negotiation
  * session resumption vs keep-alives
    * send QUIC PING frames periodically
  * Streams
    * unidirectional for Bundle delivery

-->

# Introduction

TODO Introduction


# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Operational Considerations

{{RFC9308}} references measurements indicating that a single-digit
percentage of networks block all UDP traffic. Any BP Agent initiating
a QUIC CL to a peer may encounter this kind of traffic block. While some
UDP ports might not be blocked, BP Agents SHOULD support a variety of CL
implementations and be prepared to try alternate CLs in the presence
of CL-level failures.

Similarly, BP Agents that offer a listening QUIC CL endpoint SHOULD
support a variety of CL implementations and be prepared to receive
other CL connections to support clients in challenged networks.

A list of recommended CLs and any ordering of preference for their use
is out of scope of this document.

# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
