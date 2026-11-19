---
title: "Best Practices for L4S Implementation for Mobile Devices"
abbrev: "Mobile L4S"
category: bcp
docname: draft-sporeba-tsvwg-mobile-l4s-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: Transport
workgroup: Transport Area Working Group
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
  github: "fridek/draft-sporeba-tsvwg-mobile-l4s"
  latest: "https://fridek.github.io/draft-sporeba-tsvwg-mobile-l4s/draft-sporeba-tsvwg-mobile-l4s.html"

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
  RFC9331:
  RFC9332:
  RFC9768:
  RFC9956:

informative:
  RFC3168:
  RFC9330:
  IEEE-802.11-2024:
    target: "https://doi.org/10.1109/IEEESTD.2025.10979691"
    title: "IEEE Standard for Information Technology--Telecommunications and Information Exchange between Systems - Local and Metropolitan Area Networks--Specific Requirements - Part 11: Wireless LAN Medium Access Control (MAC) and Physical Layer (PHY) Specifications"
    author:
      - org: "IEEE"
    date: 2025-04
    seriesinfo:
      IEEE: "Std 802.11-2024"
      DOI: "10.1109/IEEESTD.2025.10979691"
  TS38.214:
    target: "https://www.3gpp.org/dynareport/38214.htm"
    title: "NR; Physical layer procedures for data"
    author:
      - org: "3GPP"
    seriesinfo:
      3GPP: "TS 38.214"
  TS36.213:
    target: "https://www.3gpp.org/dynareport/36213.htm"
    title: "Evolved Universal Terrestrial Radio Access (E-UTRA); Physical layer procedures"
    author:
      - org: "3GPP"
    seriesinfo:
      3GPP: "TS 36.213"
  I-D.livingood-low-latency-deployment:
  RFC10055:
  IMC16-QCUT:
    target: "https://doi.org/10.1145/2987443.2987490"
    title: "Understanding On-device Bufferbloat for Cellular Upload"
    author:
      - ins: Y. Guo
        name: Yihua Guo
      - ins: F. Qian
        name: Feng Qian
      - ins: Q. A. Chen
        name: Qi Alfred Chen
      - ins: Z. M. Mao
        name: Zhuoqing Morley Mao
      - ins: S. Sen
        name: Subhabrata Sen
    date: 2016-11
    seriesinfo:
      ACM: "Proceedings of the 2016 Internet Measurement Conference (IMC '16)"
      DOI: "10.1145/2987443.2987490"

...
--- abstract

This document describes best practices for deployment of Low Latency, Low Loss, and Scalable Throughput (L4S) in mobile devices. It defines the responsibilities of the host operating system and the link-layer (modem and Wi-Fi) subsystems to ensure successful end-to-end low-latency communication.

--- middle

# Introduction

L4S (Low Latency, Low Loss, Scalable Throughput) {{RFC9330}} offers a framework to significantly reduce queueing delay and loss while maintaining consistent throughput. Mobile devices often have to react to quickly changing connectivity conditions and may be subject to variable throughput and connection quality. This can cause large variations in user-perceived latency and greater bufferbloat than in other devices. Deploying L4S in a mobile ecosystem requires co-operation across multiple layers: the network stack, the host operating system (OS), link-layer drivers and firmware (e.g., Wi-Fi and cellular modem), and the network. This document outlines best current practices for the mobile device subsystems to achieve reliable, low-latency performance in the field.

## Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses the four codepoint names of the 2-bit IP-ECN field defined in {{RFC3168}} and Section 1.3 of {{RFC9332}}: `Not-ECT`, `ECT(0)`, `ECT(1)`, and `CE`. In addition, the following collective terms are used:

ECN-capable packet:
: An IP packet carrying `ECT(0)`, `ECT(1)`, or `CE` in the IP-ECN field (that is, any codepoint other than `Not-ECT`).

L4S identifier (and L4S packet):
: An IP-ECN field set to `ECT(1)` or `CE`, which link-layer and network nodes use to classify packets as L4S packets without per-flow state (Section 5.1 of {{RFC9331}}, Section 2.3 of {{RFC9332}}). Note that sending endpoints mark outgoing L4S packets as `ECT(1)`, whereas `CE` indicates congestion experienced along the path.

# Host Operating System Requirements

The host operating system controls application-level network access and hosts the primary TCP and UDP transport stacks.

## Socket APIs for UDP

To enable UDP-based transport stacks (such as QUIC) to utilise L4S, the OS MUST provide APIs that allow applications to:

1. Set the ECN codepoint {{RFC9331}} on outgoing packets.
1. Read the ECN codepoints of incoming packets.

These capabilities MUST be exposed via standard socket APIs (e.g., `IP_TOS` and `IPV6_TCLASS` for setting, and `IP_RECVTOS` and `IPV6_RECVTCLASS` via ancillary data for reading) and MUST NOT be restricted by default security policies for standard application sockets. The APIs MUST allow reading the ECN codepoints on a per-packet basis, and MUST allow setting the ECN codepoints either on a per-socket or per-packet basis. The detailed explanation of configuring UDP sockets for ECN for common platforms is covered in {{RFC10055}}. It should be noted that on some platforms, setting ECN codepoints shares calls with DSCP values, and applications might need to take steps to avoid overwriting existing DSCP or ECN markings.

UDP-based transport stacks SHOULD mark L4S-capable traffic as `ECT(1)` and MUST NOT mark non-L4S traffic as `ECT(1)`. Other Non-Queue-Building (NQB) traffic (e.g., marked with DSCP 45) is considered compatible with L4S and can share the low-latency queue (Section 3.3 of {{RFC9956}}), and MUST NOT be marked as `ECT(1)` unless it also satisfies the Prague L4S requirements {{RFC9331}}. Link-layer handling of misbehaving traffic is described in {{defense-against-misbehaving-traffic}}.

## TCP Support Detection and Bootstrapping

To allow L4S-capable senders (e.g., Internet servers) to take advantage of L4S, host operating systems MUST negotiate Accurate ECN (AccECN) support and verify path integrity as described in Section 3 of {{RFC9768}}.

### Per-Network Detection and Latency Mitigation

If misconfigured networks or servers drop ECN negotiation packets, and the client does not retry without ECN negotiation, TCP connections will fail. Therefore, clients MUST implement retry strategies that disable ECN negotiation, as described in Section 3.1.4.1 of {{RFC9768}}. Note that a SYN-ACK declining AccECN or ECN in the TCP header flags (falling back to Classic ECN or Non-ECN as per Section 3.1.2 of {{RFC9768}}) does not require any retries and is not considered a failure.

Latency and loss are critical to mobile applications, and retry strategies dependent on retransmissions and timeouts can lead to a degraded user experience. To ensure that L4S can be safely enabled without degrading the user experience in the presence of networks or servers that drop ECN negotiations, Accurate ECN TCP options, or `ECT(1)`-marked packets, the client SHOULD balance implementing the retry strategies in Section 3 of {{RFC9768}} with mechanisms to reduce the latency impact of retransmissions. A possible strategy is to only attempt ECN negotiation once on the initial SYN (disabling it immediately upon retransmission). Additionally, the host MAY cache ECN negotiation or `ECT(1)` timeouts on a per-host, per-IP-address, or other basis so subsequent connections to servers known not to respond skip ECN negotiation.

If a host is in a network that blocks all ECN negotiation regardless of destination, all TCP connections will suffer latency impact. This will degrade the user experience even if failures are cached, because any connection to a server that is not in the cache will suffer an additional round-trip delay.

A host system that wants to be resilient to this MAY attempt a connectivity check to a known, L4S-supporting service. In case of check failure, the result can be used to turn off ECN negotiation attempts for a given network, represented by PLMN/APN (in carrier networks) or BSSID (in Wi-Fi networks).

When maintaining per-destination or per-network caches, entries SHOULD be retired after a reasonable TTL (e.g., 7 days).

# Link-Layer Subsystem Requirements

Link-layer subsystems, such as the cellular modem and Wi-Fi, schedule and manage transmission over the physical medium. Measurement studies on mobile devices have shown that uplink standing queues form inside link-layer driver and baseband firmware buffers rather than in the host operating system networking stack, which renders host-level queue management ineffective on its own {{IMC16-QCUT}}. Moreover, because grant-based cellular links rely on buffer status reports derived from modem-resident queues to request uplink radio resources, queueing cannot simply be pushed up into the host operating system without degrading throughput. Consequently, the low-latency queue and active queue management MUST be implemented within the link-layer subsystem itself.

## Link-Layer Inbound Packet Reordering

Some link layers provide strong ordering guarantees for inbound packets by assigning a link-layer sequence number to each packet and buffering incoming packets so they can be presented to the host operating system in order. In particular, cellular networks often perform out-of-order packet delivery at the physical layer, but require the cellular modem to deliver received packets to the host operating system in order. When packets are received out of order on the air interface, the modem waits up to a network-configurable timeout to receive all previous packets. This causes out-of-order packets to be delayed until all previous packets have been received, and causes latency spikes when packets are lost due to transmission errors.

Delaying received packets increases latency, which is contrary to the low-latency goals of L4S. Also, by artificially introducing delays that were not imposed by the network, it reduces the accuracy of protocol rate estimation. The additional delays do not benefit protocol stacks which are equipped to handle out-of-order packets.

Link-layer subsystems MUST NOT buffer outbound and inbound ECN-capable packets in a way that imposes measurable latency to the protocol stack. Correspondingly, cellular modems SHOULD ensure that ECN-capable packets can be delivered out-of-order rather than delay to deliver them in-order. This ensures that air-interface retransmissions do not cause head-of-line blocking stalls.

Section 4.3 of {{RFC9331}} specifies that loss detection SHOULD be resilient to packet reordering. However, to enable the requirement on the cellular modem behaviour described above, L4S-aware protocol stacks on mobile devices MUST be prepared to receive out-of-order packets.

## Uplink Active Queue Management (AQM) {#uplink-aqm}

The link-layer uplink buffer within mobile subsystems (such as cellular modems and Wi-Fi drivers) operates as a dynamic bottleneck subject to rapid capacity fluctuations driven by radio grant scheduling, carrier aggregation shifts, and physical layer channel fading. To achieve consistently low latency under these conditions without inducing throughput degradation, implementations MUST deploy a Dual-Queue Coupled AQM complying with the functional requirements in Section 2.5.1 of {{RFC9332}}. In particular, the subsystem MUST maintain separate queues for low-latency and Classic traffic, each managed by an AQM, where the low-latency queue AQM applies `CE` marking to ECN-capable packets and a work-conserving scheduler grants conditional priority to the low-latency queue without starving Classic traffic (Section 4.2.2 of {{RFC9332}}).

As mentioned in Notes a. to Appendix A.1 of {{RFC9332}}, the drain rate of the queue can vary if it is scheduled relative to other queues or if it accommodates fluctuations in a wireless medium. To auto-adjust to changes in drain rate, the queue needs to be measured in time (e.g., packet sojourn time), not bytes or packets. This approach naturally accommodates large aggregated frame structures (e.g., up to 64kB aggregated packets passed via GSO) without triggering premature congestion signals.

### Wi-Fi Subsystem Sojourn Time Recommendations

In a shared wireless medium such as Wi-Fi, transmission requires contention-based spectrum acquisition and relies on frame aggregation to achieve channel efficiency. Often a buffer once scheduled for transmission cannot be expanded, resulting in traffic bursts and a potential delay for subsequent packets.

In low-congestion networks, channel access is not a bottleneck, so L4S will work as designed on the bottleneck link. To prevent erroneously signalling congestion during normal frame aggregation and transmission bursts, Wi-Fi implementations SHOULD configure the sojourn time step threshold to longer than the maximum burst time. Because modern Wi-Fi standards {{IEEE-802.11-2024}} bound an A-MPDU burst by both the maximum A-MPDU size and a maximum airtime of approximately 5.5 ms, a sojourn time threshold of 6 ms is a reasonable choice.

When the latency is dominated by the channel acquisition time, `CE` marking might be counterproductive because it would signal congestion that cannot be resolved by the client backing off. In these cases, Wi-Fi firmware SHOULD NOT mark congestion.

In standards that support multi-Traffic-Identifier (multi-TID) frame aggregation (such as IEEE 802.11be), different access categories can be mixed into a single A-MPDU aggregate. In these scenarios, the link-layer subsystem SHOULD NOT bundle latency-sensitive L4S frames with traffic using longer transmission windows.

### Cellular Subsystem Sojourn Time Recommendations

Unlike the contention-based channel access of Wi-Fi, cellular networks utilise centralised, grant-based uplink scheduling managed by the base station.

During sustained transmission, the base station allocates uplink resources based on buffer status reports in ongoing traffic. Without contention backoff, the queue drains on each radio scheduling interval. Cellular modem implementations SHOULD configure the native L4S sojourn time step threshold based on the radio scheduling interval and uplink grant-to-transmission delay. For example, a threshold of 2 ms is appropriate for 5G networks {{TS38.214}}, whereas 4 ms is appropriate for 4G networks {{TS36.213}}.

When a flow transitions from an idle buffer to an active state, the device must first request uplink transmission resources from the network. This initial scheduling turnaround can introduce a delay. To prevent this initial delay from triggering spurious `CE` marks on the initial packets of a burst or connection, implementations SHOULD apply an initial burst tolerance.

Where the cellular network provides pre-allocated uplink transmission resources (such as semi-persistent scheduling or configured grants), the initial scheduling turnaround is eliminated, allowing implementations to use the lower steady-state threshold directly.

## Multi-Queue Scheduling and Bounded Latency Queueing

Any link layer that supports L4S MUST provide a low-latency queue designated for L4S packets and Non-Queue-Building traffic (as defined by Section 3.3 of {{RFC9956}}). This queue MUST be bounded as described in {{uplink-aqm}} and {{defense-against-misbehaving-traffic}}.

Some link-layer systems already support high-priority and low-priority queues within a bearer. These queues are typically not latency-bounded, and therefore cannot guarantee the low-latency benefits of L4S. If the link layer provides such queues, the low-latency queue MUST be distinct from them. An example configuration might be:

*  **Low-Latency Queue:** For L4S and other Non-Queue-Building traffic.
*  **High-Priority Queue:** For high-priority but potentially queue-building traffic.
*  **Low-Priority Queue:** For queue-building traffic (e.g., CUBIC/Reno) and bulk data.

When a Dual-Queue Coupled AQM ({{uplink-aqm}}) operates alongside high-priority and low-priority queues within the same bearer, traffic in the high-priority queue MUST NOT take unbounded precedence over the low-latency queue in a manner that delays low-latency packets beyond their target sojourn threshold. Scheduling and congestion coupling across the low-latency, high-priority, and low-priority queues SHOULD follow the guidance in Sections 2.5.1 and 4.2.2 of {{RFC9332}} so that queue-building flows remain bounded, the low-latency queue does not starve either queue-building queue under overload, and L4S flows coexist fairly with traffic in both queue-building queues.

## Packet Classification

The link layer MUST steer uplink packets carrying the L4S identifier in the IP header to the low-latency queue. The link layer SHOULD also support mapping to the low-latency queue based on the NQB DSCP value (45) {{RFC9956}} as an alternative or supplementary classifier for local uplink queueing. However, because DSCP markings are frequently bleached or remarked at network boundaries, ECN mapping remains the most reliable end-to-end classifier across mobile networks.

Where a device uses local host metadata (such as per-application or per-user priority markings) to steer traffic into a high-priority queue, classification based on the L4S identifier and NQB markings MUST take precedence so that low-latency packets from prioritised applications are not diverted into a queue-building queue.

Link-layer networks MUST NOT attempt to dynamically classify packets for the low-latency queue using heuristic traffic inference or Deep Packet Inspection (DPI). Classification MUST rely solely on the explicit packet markings set by the application endpoints. This ensures compatibility with fully encrypted payloads and aligns with the end-to-end principle and permissionless innovation, as discussed in the ISP deployment observations in {{I-D.livingood-low-latency-deployment}} (which also contains details on Wi-Fi link-layer queueing considerations).

## Defence Against Misbehaving Traffic {#defense-against-misbehaving-traffic}

Applications may erroneously or maliciously mark queue-building or unresponsive traffic with the L4S identifier without implementing responsive L4S congestion control. Because a Dual-Queue Coupled AQM operates without per-flow state, such traffic shares the low-latency queue with well-behaved flows and can drive `CE` marking to 100%, where ECN marks alone can no longer bound queueing delay or prevent ECN-capable traffic from starving `Not-ECT` flows.

Mobile link-layer subsystems MUST therefore implement the overload and unexpected-codepoint behaviours in Sections 2.5.1.1 and 4.2.3 of {{RFC9332}} (see also Section 5.1 of {{RFC9331}}). In particular, "If the DualQ Coupled AQM has detected overload, it MUST introduce Classic drop to both types of ECN-capable traffic until the overload episode has subsided" (Section 2.5.1.1 of {{RFC9332}}), and when `Not-ECT` packets are classified into the low-latency queue without separate per-flow protection, the AQM SHOULD apply Classic-appropriate drop to them when queue delay grows.

Overload drop protects overall queue stability, though a misbehaving flow can still elevate `CE` marking or loss for other low-latency flows sharing the queue. Because the device's uplink queue only carries traffic from that device (and any tethered clients), and maintaining per-flow state in modem or Wi-Fi subsystems may be difficult due to hardware limitations and ineffective across encrypted VPN tunnels (Section 4.1 of {{RFC9332}}), per-flow traffic protection (Section 5.2 of {{RFC9956}}) is OPTIONAL.

## Transparency and Bleach Prevention

The link layer MUST NOT modify the ECN field, DSCP values, or AccECN TCP options (172 and 174) on transit traffic, except for performing standard `CE` marking in the event of the queue buildup.

# Security Considerations

L4S introduces potential abuse vectors where applications mark queue-building traffic as low-latency. This risk is amplified if networks without low-latency queues instead prioritise `ECT(1)`-marked traffic using alternative heuristics (such as bandwidth allocation multipliers) without an active end-to-end congestion-marking feedback loop, incentivising endpoints to mark `ECT(1)` without implementing L4S congestion control. Regardless of network behaviour, host stacks and applications MUST NOT mark traffic as `ECT(1)` unless using compliant L4S congestion control. As described in {{defense-against-misbehaving-traffic}}, the link-layer subsystem MUST implement overload safeguards to defend against starvation and unbounded queueing delay.

# IANA Considerations

This document has no IANA actions.

--- back
