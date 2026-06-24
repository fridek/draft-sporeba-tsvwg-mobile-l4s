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
 -
    fullname: Sandeep Irlanki
    organization: Samsung R&D
    email: irlanki.s@samsung.com

normative:
  RFC3168:
  RFC6679:
  RFC8311:
  RFC8888:
  RFC9000:
  RFC9330:
  RFC9331:
  RFC9768:
  RFC9956:

informative:
  I-D.livingood-low-latency-deployment:

...
--- abstract

This document describes practical deployment considerations for Low Latency, Low Loss, and Scalable Throughput (L4S) in mobile devices. It defines the responsibilities of the host operating system, the link-layer (modem and WiFi) subsystems to ensure successful end-to-end low-latency communication.

--- middle

# Introduction

L4S (Low Latency, Low Loss, Scalable Throughput) {{RFC9330}} offers a framework to significantly reduce queuing delay while maintaining high throughput. Mobile devices often have to react to quickly changing connectivity conditions and may be subject to variable throughput and connection quality. This can cause large variations in user-perceived latency and greater bufferbloat than in other devices. This document outlines best current practices for implementing L4S in mobile devices.

Deploying L4S in a mobile ecosystem requires co-operation across multiple layers: the application, the host operating system (OS), and link-layer drivers and firmware (e.g., Wi-Fi and cellular modem), and the network. This document outlines practical deployment considerations and requirements for each of these subsystems to achieve reliable, low-latency performance in the field.


## Conventions and Definitions

{::boilerplate bcp14-tagged}

# Host Operating System Requirements

The host operating system controls application-level network access and hosts the primary TCP and UDP transport stacks.

## Socket APIs for UDP

To enable UDP-based transport stacks (such as QUIC) to utilize L4S, the OS MUST provide APIs that allow applications to:

1. Set the ECN codepoint to `ECT(1)` {{RFC9331}} on outgoing packets.
1. Read the ECN codepoints (specifically `CE` markings) of incoming packets.

These capabilities MUST be exposed via standard socket options (e.g., `IP_TOS` and `IPV6_TCLASS` for setting, and `IP_RECVTOS` and `IPV6_RECVTCLASS` via ancillary data for reading) and MUST NOT be restricted by default security policies for standard application sockets.

## TCP support detection and bootstrapping

To deploy receive-side L4S for TCP, host operating systems and link-layers must negotiate ECN support and verify path integrity.

A client that wants to be resilient to networks that don't support L4S MUST follow the negotiation steps described in {{RFC3168}}, {{RFC8311}}, {{RFC9331}}, and {{RFC9768}}
Negotiation of ECN support for TCP follows the Classic ECN setup defined in Section 6.1.1 {{RFC3168}} and recapped in Section 1.4 of {{RFC9768}}.
Negotiation of AccECN support is defined in Section 3.1 of {{RFC9768}}

These negotiations MUST include the retry mechanisms described in Section 3.1.4 and Section 3.2.3.2.2 of {{RFC9768}}.

### Per-network detection and latency mitigation

Latency can be critical to mobile applications, and fallback paths dependent on retransmissions and timeouts can lead to a degraded user experience in flows where L4S fails to be negotiated in any of the steps listed in Section 2.2. A host system that wants to be resilient to this MAY attempt a connectivity check to a known, L4S-supporting service. In case of check failure, the result can be used to turn off L4S negotiation attempts for a given network, represented by PLMN/APN (in carrier networks) or BSSID (in Wi-Fi networks). Additionally, the host system MAY maintain additional L4S support cache on a per-host or per-IP-address, or other basis. When maintaining such lists, entries should be retired after a preferred TTL (e.g., 7 days) and preferably indexed per network to disambiguate between host and path L4S support.

## Transport-Layer Feedback Prerequisites {#transport-feedback-prerequisites}

To ensure a compliant L4S deployment, transport endpoints (hosts and servers) MUST conform to the baseline protocol prerequisites for end-to-end feedback loops (including, but not limited to, the following core implementations):

* **TCP Transports:** MUST utilize Accurate ECN (AccECN) feedback loop mechanics, processing and reflecting congestion markings using the 3-bit Accurate ECN (ACE) field in the TCP header as specified in Section 3.2.2 of {{RFC9768}}. The implementation MAY also support the AccECN TCP Options specified in Section 3.2.3 of {{RFC9768}}.
* **UDP-Based Real-Time Transports:** Applications or media frameworks utilizing RTP over UDP (such as WebRTC implementations) MUST implement high-fidelity feedback loops utilizing the per-RTP packet congestion control feedback report format specified in Section 3.1 of {{RFC8888}} alongside the session negotiation and signaling parameters specified in {{RFC6679}}. The receiver MUST generate and transmit the RTP/AVPF ECN feedback packet format specified in Section 5.1 of {{RFC6679}} back to the sender, and send the RTCP XR summary report block specified in Section 5.2 of {{RFC6679}}.
* **QUIC Transports:** MUST utilize the native, fine-grained ECN feedback loops built into the IETF QUIC transport protocol layer as specified in {{RFC9000}}.
* **Other Transport Protocols:** Other transport layers (such as SCTP or DCCP) MUST utilize a fine-grained, protocol-compliant feedback loop mechanism capable of accurately reporting the extent of congestion markings back to the sender as specified in Section 4.2 of {{RFC9331}}.

# Link-layer Subsystems Requirements

The link-layer (modem and WiFi) subsystems manage the link-layer transmission over the radio interface and perform significant queueing on the uplink path.

## Link-layer inbound packet reordering

Some link-layers provide strong ordering guarantees for inbound packets by assigning a link-layer sequence number to each packet and buffering incoming packets so they can be presented to the host operating system in order. In particular, out-of-order packet delivery is common in cellular networks due to multi-path transmission. This causes out-of-order packets to be delayed until all previous packets have been received, and causes latency spikes when packets are lost due to transmission errors.

Delaying received packets increases latency, which is contrary to the low-latency goals of L4S. Also, by artificially introducing delays that were not imposed by the network, it will reduce the accuracy of protocol rate estimation. Many contemporary protocol stacks are generally well-equipped to handle out-of-order packets.

L4S-aware protocol stacks MUST be prepared to receive out-of-order packets.

Link-layers MUST NOT buffer inbound packets in a way that imposes measurable latency to the protocol stack.

## Multi-Queue Scheduling and Bounded Latency
Any link layer that supports L4S MUST support a low-latency queue designated for Non-Queue-Building {{RFC9956}} traffic. This queue MUST be bounded as described in Section 3.4 and Section 3.5.
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

## Mitigation Against Non-Compliant Prioritization

Network infrastructure nodes MUST NOT act on `ECT(1)` flags to prioritize traffic in alternative, non-compliant ways unless a valid end-to-end feedback loop is actively maintained—where the network nodes execute compliant congestion marking and the transport endpoints record and reflect those markings in line with the transport-specific requirements specified in Section 4.2 of {{RFC9331}}.

Modern network deployments MUST NOT be marketed or operated as supporting L4S if they prioritize traffic via alternative heuristics (such as bandwidth allocation multipliers) without enforcing or verifying true, end-to-end transport-layer feedback loop compliance as detailed in (#transport-feedback-prerequisites).

# Security Considerations

L4S introduces potential abuse vectors where applications mark queue-building traffic as low-latency. As described in Section 3.5, the baseband/modem subsystem MUST deploy queue protection mechanisms to defend the low-latency queue from starvation and latency degradation.

# IANA Considerations

This document has no IANA actions.

--- back
