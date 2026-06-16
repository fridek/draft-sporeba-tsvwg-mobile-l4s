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
  RFC3168:
  RFC3234:
  RFC9330:
  RFC9331:
  RFC9768:
  RFC9956:

informative:
  I-D.livingood-low-latency-deployment:

...
--- abstract

This document describes practical deployment considerations for Low Latency, Low Loss, and Scalable Throughput (L4S) in mobile networks. It defines the responsibilities of the host operating system, the link-layer (modem and WiFi) subsystems to ensure successful end-to-end low-latency communication.

--- middle

# Introduction

L4S (Low Latency, Low Loss, Scalable Throughput) {{RFC9330}} offers a framework to significantly reduce queuing delay while maintaining high throughput. Mobile devices often have to react to quickly changing connectivity conditions and may be subject to variable throughput and connection quality. This can cause large variations in user-perceived latency and greater bufferbloat than fixed devices. This document outlines best current practices for implementing L4S in mobile devices.

Deploying L4S in a mobile ecosystem requires co-operation across multiple layers: the application, the host operating system (OS), and link-layer drivers and firmware (e.g., Wi-Fi and cellular modem), and the core network middleboxes {{RFC3234}}. This document outlines practical deployment considerations and requirements for each of these subsystems to achieve reliable, low-latency performance in the field.


## Conventions and Definitions

{::boilerplate bcp14-tagged}

# Host Operating System Requirements

The host operating system controls application-level network access and hosts the primary TCP and UDP transport stacks.

## TCP Accurate ECN (AccECN) and Fallback

The host OS kernel TCP stack SHOULD support the Accurate Explicit Congestion Notification {{RFC9768}} and an L4S-compatible congestion control algorithm (e.g., TCP Prague).
To defend against middleboxes that drop SYN packets containing ECN or AccECN options, the client TCP stack SHOULD implement a fallback mechanism: if the initial SYN packet containing ECN/AccECN options times out or is dropped, the stack SHOULD attempt to negotiate AccECN at least one more time on the first retransmission. If the first retransmission also fails to be acknowledged, subsequent retransmissions SHOULD fall back by clearing the ECN and AccECN options, as specified in Section 3.1.4.1 of {{RFC9768}}.


## AccECN Path Traversal and Option Handling

During the TCP three-way handshake, AccECN capability is negotiated solely using TCP header flags (the AE, CWR, and ECE flags in the SYN and SYN/ACK), as specified in Section 3.1 of {{RFC9768}}. However, once the connection is established, Detailed Feedback is exchanged using the AccECN TCP Option (Kind 172 or 174).

Because some network paths may block, drop, or strip packets containing unrecognized TCP options, host OS TCP stacks MUST implement the following path traversal and fallback mechanisms:

*  **Detection during Handshake**: If a TCP Server sends a SYN/ACK containing the AccECN option and does not receive an ACK (either due to packet loss or middlebox blocking), the server SHOULD retransmit the SYN/ACK without the AccECN option on the first retry. If this retransmission also times out, the server SHOULD retransmit the SYN/ACK with ECN flags cleared (AE,CWR,ECE) = (0,0,0) and no option, as specified in Section 3.2.3.2.2 of {{RFC9768}}.
*  **Option Traversal Monitoring**: The host TCP stack MUST actively monitor the traversal of packets containing the AccECN option. If the stack detects a high drop rate on packets carrying the option compared to optionless packets, it MUST disable the use of the AccECN option for the remainder of the half-connection (falling back to the TCP header `ACE` field for ECN feedback).
*  **AccECN Option Usage**: The host TCP stack MUST NOT include the AccECN option on every segment. The option SHOULD only be included on scheduled ACKs when one or more byte counters have changed since the previous ACK (Section 3.2.3.3 of {{RFC9768}}). If the available TCP option space is constrained (e.g., due to SACK options for packet loss, which MUST take priority), the AccECN option MUST be truncated or omitted entirely.
*  **Fallback to Header Flags**: When the AccECN option is stripped or disabled due to detected drops, ECN feedback continues to operate gracefully using only the standard 3-bit `ACE` field in the TCP header flags. Senders MUST be capable of extracting ECN feedback from the `ACE` field using the packet-to-byte estimation algorithm defined in Appendix A.3 of {{RFC9768}}.


## Socket APIs for UDP

To enable userspace transport stacks (such as QUIC and WebRTC) to utilize L4S, the OS MUST provide APIs that allow applications to:

1. Set the ECN codepoint to `ECT(1)` {{RFC9331}} on outgoing packets.
1. Read the ECN codepoints (specifically `CE` markings) of incoming packets.

These capabilities MUST be exposed via standard socket options (e.g., `IP_TOS` and `IPV6_TCLASS` for setting, and `IP_RECVTOS` and `IPV6_RECVTCLASS` via ancillary data for reading) and MUST NOT be restricted by default security policies for standard application sockets.

# Link-layer Subsystems Requirements

The link-layer (modem and WiFi) subsystems manage the link-layer transmission over the radio interface and perform significant queueing on the uplink path.

## Link-layer inbound packet reordering

Some link-layers provide strong ordering guarantees for inbound packets by assigning a link-layer sequence number to each packet and buffering incoming packets so they can be presented to the host operating system in order. In particular, out-of-order packet delivery is common in cellular networks due to multi-path transmission. This causes out-of-order packets to be delayed until all previous packets have been received, and causes latency spikes when packets are lost due to transmission errors.

Delaying received packets increases latency, which is contrary to the low-latency goals of L4S. Also, by artificially introducing delays that were not imposed by the network, it will reduce the accuracy of protocol rate estimation. Many contemporary protocol stacks are generally well-equipped to handle out-of-order packets.

L4S-aware protocol stacks MUST be prepared to receive out-of-order packets.

Link-layers MUST NOT buffer inbound packets in a way that imposes measurable latency to the protocol stack.

## Multi-Queue Scheduling and Bounded Latency
Any link layer that supports L4S MUST support a low-latency queue designated for Non-Queue-Building {{RFC9956}} traffic. This queue MUST be bounded as described in section 3.4.
Some modem systems are known to already support high-priority and low-priority queues. These queues are typically not latency-bounded, so in the presence of such queues, low-latency queue MUST be distinct from them. An example configuration might be:


*  **L4S/Low-Latency Queue:** For `ECT(1)` and DSCP-45 marked traffic.
*  **High-Priority Queue:** For signaling, IMS voice (VoLTE/VoNR), and other critical real-time traffic.
*  **Low-Priority Queue:** For queue-building traffic (e.g., CUBIC/Reno) and bulk data.

The scheduler MUST prioritize the Low-Latency Queue, but SHOULD use a scheduling algorithm (e.g., Weighted Fair Queueing) that prevents starvation of other queues.

## Packet Classification

The link-layer MUST map uplink traffic to the low-latency queue based on ECN markings:

*  Packets carrying the `ECT(1)` or `CE` bits in the IP header MUST be steered to the low-latency queue.
*  The modem SHOULD also support mapping to the low-latency queue based on the Non-Queue-Building (NQB) DSCP value (45) {{RFC9956}} as an alternative or supplementary classifier. Because DSCP markings are frequently bleached at carrier interconnect boundaries, ECN mapping remains the most reliable end-to-end classifier for mobile networks.

Link-layer networks MUST NOT attempt to dynamically classify packets for the low-latency queue using heuristic traffic inference or Deep Packet Inspection (DPI). Classification MUST rely solely on the explicit packet markings set by the application endpoints. This ensures compatibility with fully encrypted payloads and aligns with the end-to-end principle and permissionless innovation, as discussed in the ISP deployment observations in {{I-D.livingood-low-latency-deployment}} (which also contains details on Wi-Fi link-layer queuing considerations).


## Uplink Active Queue Management (AQM)
The link-layer uplink buffer is often a bottleneck due to cellular grant scheduling. When the uplink queue builds up, the modem MUST perform ECN marking:

*  If the sojourn time of a packet in the L4S queue exceeds a shallow threshold (e.g., 1 ms to 5 ms), the modem MUST mark the packet as `CE` in the IP header before transmitting it, rather than dropping it.
*  Packets MUST only be dropped if the queue reaches the maximum designated size.

## Defense Against Misbehaving Traffic (Queue Protection)
Applications may mark their traffic as NQB or `ECT(1)` without implementing L4S congestion control, causing queue build-up in the low-latency queue.

*  The modem MUST enforce a strict size limit on the Low-Latency Queue (e.g. 16kB). If the queue is full, incoming packets MUST be dropped.
*  The modem SHOULD monitor queue build-up and latency contributions of individual flows within the L4S queue.
*  If a flow is detected to be queue-building (e.g., contributing to sustained queue latency above the marking threshold without responding to CE marks), the modem MUST demote the flow and redirect its packets to a different queue.

## Transparency and Bleach Prevention
The modem MUST NOT modify the ECN bits, DSCP flags, or AccECN TCP options (172 and 174) on low-latency-queue transit traffic, except for performing standard CE marking when congested.

# Middlebox Requirements

Middleboxes include cellular core network elements (such as the UPF and PGW), firewalls, NATs, and deep packet inspection (DPI) appliances.

Middleboxes MUST NOT perform network-based classification or rewrite ECN/DSCP markings based on traffic heuristics or DPI. In accordance with {{I-D.livingood-low-latency-deployment}}, active classification decisions MUST be left to the application endpoints, and middleboxes MUST restrict their role to passive, transparent forwarding.

## ECN and AccECN Transparency
Middleboxes MUST NOT clear (bleach) ECN bits, in accordance with {{RFC3168}}. They MUST preserve `ECT(0)`, `ECT(1)`, and `CE` markings on all IP packets.
Furthermore, middleboxes MUST NOT strip, modify, or drop packets containing TCP options 172 or 174.

## Handshake Forwarding
Middleboxes MUST transparently forward `SYN` and `SYN-ACK` packets that negotiate ECN or AccECN. Middleboxes MUST NOT drop TCP handshake packets solely due to the presence of ECN negotiation flags or AccECN TCP options.

# Security Considerations

L4S introduces potential abuse vectors where applications mark queue-building traffic as low-latency. As described in Section 3.5, the baseband/modem subsystem MUST deploy queue protection mechanisms to defend the low-latency queue from starvation and latency degradation.

# IANA Considerations

This document has no IANA actions.

--- back
