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
    email: sporeba@google.com
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
  RFC8311:
  RFC9330:
  RFC9331:
  RFC9768:
  RFC9956:

informative:
  I-D.livingood-low-latency-deployment:
  I-D.ietf-tsvwg-udp-ecn:

...
--- abstract

This document documents best practices for deployment of Low Latency, Low Loss, and Scalable Throughput (L4S) in mobile devices. It defines the responsibilities of the host operating system, the link-layer (modem and WiFi) subsystems to ensure successful end-to-end low-latency communication.

--- middle

# Introduction

L4S (Low Latency, Low Loss, Scalable Throughput) {{RFC9330}} offers a framework to significantly reduce queuing delay while maintaining high throughput. Mobile devices often have to react to quickly changing connectivity conditions and may be subject to variable throughput and connection quality. This can cause large variations in user-perceived latency and greater bufferbloat than in other devices. Deploying L4S in a mobile ecosystem requires co-operation across multiple layers: the network stack, the host operating system (OS), and link-layer drivers and firmware (e.g., Wi-Fi and cellular modem), and the network. This document outlines best current practices for each of these subsystems to achieve reliable, low-latency performance in the field.

## Conventions and Definitions

{::boilerplate bcp14-tagged}

# Host Operating System Requirements

The host operating system controls application-level network access and hosts the primary TCP and UDP transport stacks.

## Socket APIs for UDP

To enable UDP-based transport stacks (such as QUIC) to utilize L4S, the OS MUST provide APIs that allow applications to:

1. Set the ECN codepoint {{RFC9331}} on outgoing packets.
1. Read the ECN codepoints of incoming packets.

These capabilities MUST be exposed via standard socket APIs (e.g., `IP_TOS` and `IPV6_TCLASS` for setting, and `IP_RECVTOS` and `IPV6_RECVTCLASS` via ancillary data for reading) and MUST NOT be restricted by default security policies for standard application sockets. The APIs MUST allow reading the ECN codepoints on a per-packet basis. and MUST allow setting the ECN codepoints either on a per-socket or per-packet basis. The detailed explanation of configuring UDP sockets for ECN for common platforms is covered in {{I-D.ietf-tsvwg-udp-ecn}}.

UDP-based transport stacks SHOULD mark L4S-capable traffic as ECT(1) and MUST NOT mark queue-building traffic (e.g., traffic using legacy congestion controls) as ECT(1). Link layers MUST respond to misbehaving stack as discussed in {#defense-against-misbehaving-traffic}.

## TCP support detection and bootstrapping

To allow L4S-capable senders (e.g., Internet servers) to take advantage of L4S, host operating systems and link-layers must negotiate ECN support and verify path integrity as described in Section 1.4 and Section 3 of {{RFC9768}}.

### Per-network detection and latency mitigation

If misconfigured networks or servers drop ECN negotiation packets, and the client does not retry without ECN negotiation, TCP connections will fail. Therefore, clients MUST implement retry strategies that disable ECN negotiation, as described in Section 3.1.4 and Section 3.2.3.2.2 of {{RFC9768}}. Note that a server replying with Not-ECT is does not require any additional retries and is not considered a failure.

Latency is critical to mobile applications, and retry strategies dependent on retransmissions and timeouts can lead to a degraded user experience. To ensure that L4S can be safely enabled without degrading the user experience in the presence of networks or servers that drop ECN negotiations or Accurate ECN TCP options, the client SHOULD balance implementing the retry strategies in section 3 of {{RFC9768}} with mechanisms to reduce the latency impact of retransmissions. A possible strategy is to only attempt ECN negotiation on the first SYN to a new server. Additionally, the host MAY cache ECN negotiation timeouts on a per-host or per-IP-address, or other basis.

If a host is in a network that blocks all ECN negotiation regardless of destination, all TCP connections will suffer latency impact. This will degrade the user experience even if failures are cached, because any connection to a server that is not in the cache will suffer an additional round trip delay.

A host system that wants to be resilient to this MAY attempt a connectivity check to a known, L4S-supporting service. In case of check failure, the result can be used to turn off L4S negotiation attempts for a given network, represented by PLMN/APN (in carrier networks) or SSID/BSSID (in Wi-Fi networks).

When maintaining such lists, entries SHOULD be retired after a reasonable TTL (e.g. 7 days), and SHOULD prefer information from more frequently used destinations.

# Link-layer Subsystems Requirements

Link-layer subsystems, such as the cellular modem and Wi-Fi schedule and manage the link-layer transmission over the physical medium interface. These systems typically perform significant queueing on the transmit path, and sometimes on the receive path as well.

## Link-layer inbound packet reordering

Some link layers provide strong ordering guarantees for inbound packets by assigning a link-layer sequence number to each packet and buffering incoming packets so they can be presented to the host operating system in order. In particular, cellular networks often perform out-of-order packet delivery at the physical layer, but require the cellular modem to deliver received packets to the host operating system in order. When packets are received out of order on the air interface, the modem waits up to a network-configurable timeout to receive all previous packets. This causes out-of-order packets to be delayed until all previous packets have been received, and causes latency spikes when packets are lost due to transmission errors.

Delaying received packets increases latency, which is contrary to the low-latency goals of L4S. Also, by artificially introducing delays that were not imposed by the network, it reduces the accuracy of protocol rate estimation. The additional delays do not benefit contemporary protocol stacks, which are generally well-equipped to handle out-of-order packets.

L4S-aware protocol stacks MUST be prepared to receive out-of-order packets.

Link-layers MUST NOT buffer inbound L4S packets in a way that imposes measurable latency to the protocol stack.

## Multi-Queue Scheduling and Bounded Latency Queueing

Any link layer that supports L4S MUST provide a low-latency queue designated for L4S traffic and Non-Queue-Building traffic ({{RFC9956}} section 3.3). This queue MUST be bounded as described in {#uplink-aqm} and {#defense-against-misbehaving-traffic}.
Some link-layer systems already support high-priority and low-priority queues. These queues are typically not latency-bounded, and therefore cannot guarantee the low-latency benefits of L4S. If the link layer provides such queues, low-latency queue MUST be distinct from them. An example configuration might be:


*  **Low-Latency Queue:** For L4S and other Non-Queue-Building traffic.
*  **High-Priority Queue:** For high priority but potentially queue-building traffic.
*  **Low-Priority Queue:** For queue-building traffic (e.g., CUBIC/Reno) and bulk data.

Link layers SHOULD ensure that the L4S queue does not starve the other queues if offered L4S traffic is higher than available bandwidth. Such prioritisation SHOULD use a scheduling algorithm (e.g., Weighted Fair Queueing) and aim to minimize the queue buildup in the Low-Latency queue.

## Packet Classification

The link-layer MUST map uplink traffic to the low-latency queue based on ECN markings:

*  Packets carrying the `ECT(1)` or `CE` bits in the IP header MUST be steered to the low-latency queue.
*  The link-layer SHOULD also support mapping to the low-latency queue based on the Non-Queue-Building (NQB) DSCP value (45) {{RFC9956}} as an alternative or supplementary classifier. Because DSCP markings are frequently bleached at carrier interconnect boundaries, ECN mapping remains the most reliable end-to-end classifier for mobile networks.

Link-layer networks MUST NOT attempt to dynamically classify packets for the low-latency queue using heuristic traffic inference or Deep Packet Inspection (DPI). Classification MUST rely solely on the explicit packet markings set by the application endpoints. This ensures compatibility with fully encrypted payloads and aligns with the end-to-end principle and permissionless innovation, as discussed in the ISP deployment observations in {{I-D.livingood-low-latency-deployment}} (which also contains details on Wi-Fi link-layer queuing considerations).


## Uplink Active Queue Management (AQM) {#uplink-aqm}

The link-layer uplink buffer is often a bottleneck due to cellular grant scheduling. When the uplink queue builds up, the link-layer MUST perform ECN marking:

*  If the sojourn time of a packet in the L4S queue exceeds a shallow threshold (e.g., 1 ms to 5 ms), the link-layer MUST mark the packet as `CE` in the IP header before transmitting it, rather than dropping it.
*  Packets MUST only be dropped if the queue reaches the maximum designated size.

## Defense Against Misbehaving Traffic (Queue Protection) {#defense-against-misbehaving-traffic}

Applications may mark their traffic as NQB or `ECT(1)` without implementing L4S congestion control, causing queue build-up in the low-latency queue.

*  The link-layer MUST enforce a strict size limit on the Low-Latency Queue. If the queue is full, incoming packets MUST be dropped.
*  The link-layer SHOULD monitor queue build-up and latency contributions of individual flows within the L4S queue.
*  If a flow is detected to be queue-building (e.g. contributing to sustained queue latency above the marking threshold without responding to CE marks), the link-layer SHOULD demote the flow and redirect its packets to a different queue.

## Transparency and Bleach Prevention

The link-layer MUST NOT modify the ECN bits, DSCP flags, or AccECN TCP options (172 and 174) on low-latency-queue transit traffic, except for performing standard CE marking in the event of queue buildup.

# On-Path Node Requirements

On-Path Nodes MUST NOT perform network-based classification or rewrite ECN/DSCP markings based on traffic heuristics or DPI. In accordance with {{I-D.livingood-low-latency-deployment}}, active classification decisions MUST be left to the application endpoints, and on-path devices MUST restrict their role to marking the ECN bits in the event of a queue forming.

## ECN and AccECN Transparency

On-Path Nodes MUST NOT clear (bleach) ECN bits, in accordance with {{RFC3168}}. They MUST preserve `ECT(0)`, `ECT(1)`, and `CE` markings on all IP packets, except in the event of queue forming, where appropriate codepoints should be marked. Similarly, On-Path Nodes MUST NOT strip, modify, or drop packets containing TCP options 172 or 174.

Non-compliant behaviour can lead to timeouts and retransmits in the TCP handshake, and consequently to a degraded user experience.

## Handshake Forwarding

On-Path Nodes MUST transparently forward `SYN` and `SYN-ACK` packets that negotiate ECN or AccECN. On-Path Nodes MUST NOT drop TCP handshake packets solely due to the presence of ECN negotiation flags or AccECN TCP options.

## Mitigation Against Non-Compliant Prioritization

On-Path Nodes MUST NOT act on `ECT(1)` flags to prioritize traffic in alternative, non-compliant ways unless a valid end-to-end feedback loop is actively maintained—where the network nodes execute compliant congestion marking and the transport endpoints record and reflect those markings in line with the transport-specific requirements specified in Section 4.2 of {{RFC9331}}.

Network deployments MUST NOT be marketed or operated as supporting L4S if they prioritize traffic via alternative heuristics (such as bandwidth allocation multipliers). The risk of such alternative mechanisms is incentivising marking `ECT(1)` codepoints in flows to be allocated into prioritised queues, without implementing full L4S congestion control as described in {#defense-against-misbehaving-traffic}. The convergence point of such behaviour would be a prevalence of `ECT(1)` marked traffic that does not respond to `CE` markings and congestion in the incorrectly prioritised queues.

# Security Considerations

L4S introduces potential abuse vectors where applications mark queue-building traffic as low-latency. As described in {#defense-against-misbehaving-traffic}, the link-layer subsystem MUST deploy queue protection mechanisms to defend the low-latency queue from starvation and latency degradation.

# IANA Considerations

This document has no IANA actions.

--- back
