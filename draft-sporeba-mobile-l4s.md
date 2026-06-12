---
title: "Best Practices for L4S implementation for Mobile Devices"
abbrev: "Mobile L4S"
category: bcp
docname: draft-sporeba-mobile-l4s-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: Transport
workgroup: TSVWG Working Group
keyword:
 - L4S
 - Mobile
 - Cellular
 - ECN
 - AccECN
 - NQB
venue:
#  group: TSVWG
#  type: Working Group
  github: "fridek/draft-sporeba-mobile-l4s"
  latest: "https://fridek.github.io/draft-sporeba-mobile-l4s/draft-sporeba-mobile-l4s.html"

author:
 -
    fullname: Sebastian Poreba
    organization: Google LLC
    email: sebastian@poreba.me
 -
    fullname: Lorenzo Colitti
    organization: Google LLC
    email: lorenzo@google.com

normative:
  RFC3234:
  RFC9330:
  RFC9331:
  RFC9768:
  RFC9956:

informative:

...
--- abstract

This document describes practical deployment considerations for Low Latency, Low Loss, and Scalable Throughput (L4S) in mobile (cellular) networks. It defines the responsibilities of the Host Operating System, the Cellular and Wifi Subsystems to ensure successful end-to-end low-latency communication.

--- middle

# Introduction

Mobile devices often  have to react to quickly changing connectivity conditions and may be subject to variable throughput and connection quality.

Deploying L4S in a mobile ecosystem requires co-operation across multiple layers: the application, the host operating system (OS), the modem baseband firmware, and the core network middleboxes {{RFC3234}}. This document outlines practical deployment considerations and requirements for each of these subsystems to achieve reliable, low-latency performance in the field.

L4S (Low Latency, Low Loss, Scalable Throughput) {{RFC9330}} offers a framework to significantly reduce queuing delay while maintaining high throughput.
Networking on mobile devices frequently suffers from latency spikes due to queue build-up (bufferbloat) and hardware buffers.

## Conventions and Definitions

{::boilerplate bcp14-tagged}

# Host Operating System (OS) Requirements

The host operating system controls application-level network access and hosts the primary TCP and UDP transport stacks.

## Socket APIs for UDP/QUIC and WebRTC
To enable userspace transport stacks (such as QUIC and WebRTC) to utilize L4S, the OS MUST provide APIs that allow applications to:

1. Set the ECN codepoint to `ECT(1)` {{RFC9331}} on outgoing packets.
1. Read the ECN codepoints (specifically `CE` markings) of incoming packets.

These capabilities MUST be exposed via standard socket options (e.g., `IP_TOS` and `IPV6_TCLASS` for setting, and `IP_RECVTOS` and `IPV6_RECVTCLASS` via ancillary data for reading) and MUST NOT be restricted by default security policies for standard application sockets.

## UDP Out-of-Order Delivery
Out-of-order packet delivery is common in cellular networks due to multi-path transmission or link-layer retransmissions. Unlike TCP, UDP does not require in-order delivery at the transport layer, and applications like QUIC and WebRTC handle packet reordering in userspace.

The host OS network stack MUST NOT delay or block incoming UDP packets to enforce ordering. Enforcing in-order delivery for UDP in the OS kernel or driver introduces unnecessary Head-of-Line (HOL) blocking latency.

## TCP Accurate ECN (AccECN) and Fallback
The host OS kernel TCP stack SHOULD support AccECN negotiation {{RFC9768}} and an L4S-compatible congestion control algorithm (e.g., TCP Prague).

To defend against middleboxes that drop `SYN` packets containing ECN or AccECN options, the client TCP stack MUST implement a fallback mechanism: if the initial `SYN` packet containing ECN/AccECN options times out or is dropped, the stack MUST retransmit the `SYN` on the second attempt without ECN or AccECN options.

# Modem Subsystem Requirements

The modem (baseband) manages the link-layer transmission over the radio interface and performs significant queueing on the uplink path.

## Packet Classification
The modem MUST map uplink traffic to the low-latency queue based on ECN markings:

*  Packets carrying the `ECT(1)` or `CE` bits in the IP header MUST be steered to the low-latency queue.
*  The modem SHOULD also support mapping to the low-latency queue based on the Non-Queue-Building (NQB) DSCP value (45) {{RFC9956}} as an alternative or supplementary classifier.

## Multi-Queue Scheduling and Bounded Latency
The modem MUST support a low-latency queue designated for Non-Queue-Building {{RFC9956}} traffic. Some modem systems are known to already support high-priority and low-priority queues. In the presence of such queues, low-latency queue MUST be distinct from them.
An example configuration might be:

*  **L4S/Low-Latency Queue:** For `ECT(1)` and DSCP-45 marked traffic.
*  **High-Priority Queue:** For signaling, IMS voice (VoLTE/VoNR), and other critical real-time traffic.
*  **Low-Priority Queue:** For queue-building traffic (e.g., CUBIC/Reno) and bulk data.

The scheduler MUST prioritize the Low-Latency Queue, but SHOULD use a scheduling algorithm (e.g., Weighted Fair Queueing) that prevents starvation of other queues.

## Uplink Active Queue Management (AQM)
The modem uplink buffer is often a bottleneck due to cellular grant scheduling. When the uplink queue builds up, the modem MUST perform ECN marking:

*  If the sojourn time of a packet in the L4S queue exceeds a shallow threshold (e.g., 1 ms to 5 ms), the modem MUST mark the packet as `CE` in the IP header before transmitting it, rather than dropping it.
*  Packets MUST only be dropped if the queue reaches the maximum designated size.

## Defense Against Misbehaving Traffic (Queue Protection)
Applications may mark their traffic as NQB or `ECT(1)` without implementing L4S congestion control, causing queue build-up in the low-latency queue.

*  The modem MUST enforce a strict size limit on the Low-Latency Queue (e.g. 16kB). If the queue is full, incoming packets MUST be dropped.
*  The modem SHOULD monitor queue build-up and latency contributions of individual flows within the L4S queue.
*  If a flow is detected to be queue-building (e.g., contributing to sustained queue latency above the marking threshold without responding to CE marks), the modem MUST demote the flow and redirect its packets to a different queue.

## Transparency and Bleach Prevention
The modem MUST NOT modify the ECN bits, TCP flags, or AccECN TCP options (172 and 174) on transit traffic, except for performing standard `CE` marking when congested.

# Middlebox Requirements

Middleboxes include cellular core network elements (such as the UPF and PGW), firewalls, NATs, and deep packet inspection (DPI) appliances.

## ECN and AccECN Transparency
Middleboxes MUST NOT clear (bleach) ECN bits. They MUST preserve `ECT(0)`, `ECT(1)`, and `CE` markings on all IP packets.
Furthermore, middleboxes MUST NOT strip, modify, or drop packets containing TCP options 172 or 174.

## Handshake Forwarding
Middleboxes MUST transparently forward `SYN` and `SYN-ACK` packets that negotiate ECN or AccECN. Middleboxes MUST NOT drop TCP handshake packets solely due to the presence of ECN negotiation flags or AccECN TCP options.

# Security Considerations

L4S introduces potential abuse vectors where applications mark queue-building traffic as low-latency. As described in Section 3.4, the baseband/modem subsystem MUST deploy queue protection mechanisms to defend the low-latency queue from starvation and latency degradation.

# IANA Considerations

This document has no IANA actions.

--- back
