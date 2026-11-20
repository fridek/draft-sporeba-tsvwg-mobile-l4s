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
  RFC9768:
  RFC9956:

informative:
  RFC3168:
  RFC9000:
  RFC9330:
  RFC9332:
  IEEE-802.11-2024:
    target: "https://doi.org/10.1109/IEEESTD.2025.10979691"
    title: "IEEE Standard for Information Technology--Telecommunications and Information Exchange between Systems - Local and Metropolitan Area Networks--Specific Requirements - Part 11: Wireless LAN Medium Access Control (MAC) and Physical Layer (PHY) Specifications"
    author:
      - org: "IEEE"
    date: 2025-04
    seriesinfo:
      IEEE: "Std 802.11-2024"
      DOI: "10.1109/IEEESTD.2025.10979691"
  TS23.501:
    target: "https://www.3gpp.org/dynareport/23501.htm"
    title: "System architecture for the 5G System (5GS); Stage 2"
    author:
      - org: "3GPP"
    seriesinfo:
      3GPP: "TS 23.501"
  TS38.300:
    target: "https://www.3gpp.org/dynareport/38300.htm"
    title: "NR; NR and NG-RAN Overall description; Stage-2"
    author:
      - org: "3GPP"
    seriesinfo:
      3GPP: "TS 38.300"
  I-D.livingood-low-latency-deployment:
  I-D.ietf-intarea-reordering:
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

NQB packet (and NQB traffic):
: Non-queue building IP packet(s) with DSCP set to value `45` as defined by Section 6.1 of {{RFC9956}}.

# Mobile Device Architecture

Modern mobile devices separate the host operating system from link-layer subsystems such as cellular modems and Wi-Fi ({{fig-arch}}). The host operating system runs applications and manages socket APIs, transport protocols, and the IP stack. Link-layer subsystems manage wireless channel acquisition and air-interface retransmissions, and maintain their own internal packet buffers in both directions. Both the host operating system and link-layer subsystems need to correctly implement behaviours on the downlink and uplink paths relevant to L4S to achieve low-latency benefits.

~~~ ascii-art
+---------------- Mobile Device ----------------+
|                                               |
| +--------------+   +------------------------+ |
| |   Host OS    |   |  Link-Layer Subsystem  | |
| |              |   |   (Wi-Fi / Cellular)   | |
| | Apps, Socket |   |                        | |
| | APIs & TCP/  |   | +--------------------+ | |   Downlink
| | UDP/IP Stack |<----| Inbound Reordering |<----------------
| |              |   | |       Buffer       | | |
| |              |   | +--------------------+ | |
| |              |   | +--------------------+ | |    Uplink
| |              |---->|   Uplink Queues    |---------------->
| |              |   | |  & Channel Access  | | |
| |              |   | +--------------------+ | |
| +--------------+   +------------------------+ |
+-----------------------------------------------+
~~~
{: #fig-arch title="Mobile Device Data Path for Downlink and Uplink L4S"}

# Downlink Requirements

On the downlink, remote senders transmit L4S traffic toward the mobile device while bottleneck active queue management is performed in the network. The mobile device is responsible for delivering received packets without link-layer reordering delays, preserving congestion markings, and reporting accurate ECN feedback back to the sender.

## Host Operating System {#downlink-host-os}

### UDP Socket APIs for Reading ECN

To enable UDP-based transport stacks (such as QUIC or real-time media over RTP) to receive L4S traffic and report congestion feedback to the sender, the OS MUST provide APIs that allow applications to read the ECN codepoints of incoming packets on a per-packet basis. This capability MUST be exposed via standard socket APIs (e.g., `IP_RECVTOS` and `IPV6_RECVTCLASS` via ancillary data) and MUST NOT be restricted by default security policies for standard application sockets. Configuring UDP sockets for ECN on common platforms is detailed in {{RFC10055}}.

### TCP Support Detection and Bootstrapping {#tcp-support-detection}

To allow L4S-capable senders (e.g., Internet servers) to take advantage of L4S, host operating systems MUST negotiate Accurate ECN (AccECN) support and verify path integrity as described in Section 3 of {{RFC9768}}.

#### Per-Network Detection and Latency Mitigation {#per-network-detection}

If misconfigured networks or servers drop ECN negotiation packets, and the client does not retry without ECN negotiation, TCP connections will fail. Therefore, clients MUST implement retry strategies that disable ECN negotiation, as described in Section 3.1.4.1 of {{RFC9768}}. Note that a SYN-ACK declining AccECN or ECN in the TCP header flags (falling back to Classic ECN or Non-ECN as per Section 3.1.2 of {{RFC9768}}) does not require any retries and is not considered a failure.

Latency and loss are critical to mobile applications, and retry strategies dependent on retransmissions and timeouts can lead to a degraded user experience. To ensure that L4S can be safely enabled without degrading the user experience in the presence of networks or servers that drop ECN negotiations, Accurate ECN TCP options, or `ECT(1)`-marked packets, the client SHOULD balance implementing the retry strategies in Section 3 of {{RFC9768}} with mechanisms to reduce the latency impact of retransmissions. A possible strategy is to only attempt ECN negotiation once on the initial SYN (disabling it immediately upon retransmission). Additionally, the host MAY cache ECN negotiation or `ECT(1)` timeouts on a per-host, per-IP-address, or other basis so subsequent connections to servers known not to respond skip ECN negotiation.

If a host is in a network that blocks all ECN negotiation regardless of destination, all TCP connections will suffer latency impact. This will degrade the user experience even if failures are cached, because any connection to a server that is not in the cache will suffer an additional round-trip delay.

A host system that wants to be resilient to this MAY attempt a connectivity check to a known, L4S-supporting service. In case of check failure, the result can be used to turn off ECN negotiation attempts for a given network, represented by PLMN/APN (in carrier networks) or BSSID (in Wi-Fi networks).

When maintaining per-destination or per-network caches, entries SHOULD be retired after a reasonable TTL (e.g., 7 days).

## Link-Layer Subsystems {#downlink-link-layer}

### Inbound Packet Reordering

Some link layers provide strong ordering guarantees for inbound packets by assigning a link-layer sequence number to each frame and buffering incoming packets so they can be presented to the host operating system in order {{I-D.ietf-intarea-reordering}}. In cellular networks, physical-layer retransmissions frequently deliver packets out of order, while the modem buffers received packets up to a network-configurable timeout to deliver them in order. Similarly, in Wi-Fi {{IEEE-802.11-2024}}, when an individual frame within an aggregated A-MPDU burst is lost, BlockAck resequencing holds subsequent frames in that traffic identifier (including frames belonging to unrelated flows) until MAC contention and retransmission complete. Both mechanisms cause head-of-line blocking and latency spikes when air-interface transmission errors occur.

Delaying received packets increases latency, which is contrary to the low-latency goals of L4S. Also, by artificially introducing delays that were not imposed by the network, it reduces the accuracy of protocol rate estimation. The additional delays do not benefit protocol stacks which are equipped to handle out-of-order packets.

Link-layer subsystems MUST NOT buffer inbound ECN-capable packets in a way that imposes measurable latency to the protocol stack. Correspondingly, cellular modems and Wi-Fi subsystems MUST ensure that ECN-capable packets can be delivered out of order rather than delayed to preserve link-layer ordering, so that air-interface retransmissions do not cause head-of-line blocking stalls.

Section 4.3 of {{RFC9331}} specifies that loss detection SHOULD be resilient to packet reordering. However, to enable the link-layer behaviour described above, L4S-aware protocol stacks on mobile devices MUST be prepared to receive out-of-order packets.

### Downlink Transparency and Bleach Prevention

The link layer MUST NOT modify the ECN field, DSCP values, or AccECN TCP options (172 and 174) on inbound traffic or on outbound acknowledgements.

# Uplink Requirements

On the uplink, the mobile device originates traffic and isolates low-latency packets from queue-building traffic while acquiring wireless channel resources, relying on on-path network nodes for congestion marking.

## Host Operating System {#uplink-host-os}

### UDP Socket APIs and Packet Marking

To enable UDP-based transport stacks (such as QUIC or real-time media over RTP) to send L4S traffic, the OS MUST provide APIs that allow applications to set the ECN codepoint {{RFC9331}} on outgoing packets on either a per-socket or per-packet basis. This capability MUST be exposed via standard socket APIs (e.g., `IP_TOS` and `IPV6_TCLASS`, as detailed in {{RFC10055}}) and MUST NOT be restricted by default security policies for standard application sockets. On some platforms, setting ECN codepoints shares calls with DSCP values, and applications might need to take steps to avoid overwriting existing DSCP or ECN markings.

UDP-based transport stacks SHOULD mark L4S-capable traffic as `ECT(1)` and MUST NOT mark non-L4S traffic as `ECT(1)`. NQB traffic is considered compatible with L4S and can share the low-latency queue (Section 3.3 of {{RFC9956}}), and MUST NOT be marked as `ECT(1)` unless it also satisfies the Prague L4S requirements {{RFC9331}}. Link-layer handling of misbehaving traffic is described in {{defense-against-misbehaving-traffic}}.

Because some network paths drop or bleach ECN-marked UDP packets, UDP-based transport stacks SHOULD validate ECN path traversal and fall back to `Not-ECT` if `ECT(1)` packets are persistently lost or bleached (for example, as specified for QUIC in Section 13.4.2 of {{RFC9000}}).

### TCP Sender Requirements

Where the host OS TCP stack sends L4S traffic on the uplink it MUST use a compliant L4S congestion control algorithm {{RFC9331}}.

## Link-Layer Subsystems {#uplink-link-layer}

Link-layer subsystems, such as the cellular modem and Wi-Fi, schedule and manage transmission over the physical medium. Measurement studies on mobile devices have shown that bufferbloat forms inside link-layer driver and baseband firmware buffers rather than in the host operating system networking stack {{IMC16-QCUT}}. Moreover, because grant-based cellular links rely on buffer status reports derived from modem-resident queues to request uplink radio resources, queueing cannot simply be pushed up into the host operating system without degrading throughput. Consequently, uplink queue isolation MUST be implemented within the link-layer subsystem. The host operating system MAY also maintain a separate low-latency queue where link-layer buffering is kept shallow.

### Avoiding On-Device CE Marking {#uplink-ce-marking}

The host operating system and link-layer subsystems MUST NOT apply `CE` congestion markings to outbound uplink packets. Instead, uplink congestion marking is left to on-path network nodes, while the mobile device isolates low-latency packets from local queue-building traffic ({{low-latency-queueing}}).

In 5G cellular networks supporting L4S {{TS23.501}} {{TS38.300}}, uplink congestion detection and `CE` marking are performed in the radio access network (base station), which schedules uplink grants across radio bearers and across devices using real-time radio and cell-load metrics. Performing `CE` marking on the mobile device is redundant on such networks and risks premature `CE` marks during normal grant-scheduling turnaround delays, while host-level active queue management is ineffective because packets drain rapidly into modem buffers to populate buffer status reports {{IMC16-QCUT}}.

In shared contention-based wireless media such as Wi-Fi {{IEEE-802.11-2024}}, transmission opportunities depend on the channel acquisition across contending stations. When delay is caused by channel contention rather than an local sending rate, marking `CE` on the device is counterproductive. In this scenario backing off does not improve transmission window opportunities and places the mobile device at a throughput disadvantage relative to other contending stations.

### Low-Latency Queueing and Bearer Isolation {#low-latency-queueing}

On the uplink, the device needs to prevent Classic queue-building flows from delaying L4S and NQB packets.

In cellular networks supporting L4S (such as 5G {{TS23.501}} {{TS38.300}}), the network can assign L4S traffic to a dedicated data radio bearer - for example, dynamically upon detecting L4S packets, or via session policy - and provision classification rules on the device so that matching uplink packets are steered onto that bearer. Because each data radio bearer maintains its own link-layer queue under network-controlled scheduling, this bearer separation isolates L4S traffic from Classic queue buildup without requiring additional local queue separation in the modem. Cellular modems MUST support matching on the 2-bit IP-ECN field in network-signalled packet filters, and the host-to-modem data path MUST ensure that backpressure from a congested Classic bearer does not cause head-of-line blocking for packets destined for a dedicated L4S bearer.

Where low-latency traffic is not isolated onto a dedicated radio bearer by the network (such as on Wi-Fi links, or on shared cellular bearers where the network does not support L4S), a link-layer subsystem that supports L4S MUST provide a dedicated low-latency queue for L4S and NQB traffic. Some link-layer systems already support high-priority and low-priority queues within a shared bearer. Because such high-priority queues are typically not latency-bounded, any local low-latency queue MUST remain distinct from them:

*  **Low-Latency Queue:** For L4S and NQB traffic.
*  **High-Priority Queue:** For high-priority but potentially queue-building traffic.
*  **Low-Priority Queue:** For queue-building traffic (e.g., CUBIC/Reno) and bulk data.

Traffic in the high-priority queue MUST NOT take unbounded precedence over the low-latency queue in a manner that introduces queueing delay to low-latency packets.

The host operating system MAY also provide a low-latency queue where link-layer driver and firmware buffering is bounded (for example, to one or two transmission opportunities).

Because the device does not perform `CE` marking ({{uplink-ce-marking}}), all low-latency queues on device MUST follow the shallow-buffer and scheduling principles in Section 5.1 of {{RFC9956}}: they MUST enforce a shallow buffer (for example, 10 ms or less at the egress rate) and MUST use a work-conserving scheduler with a bounded weight (such as Deficit Round Robin) rather than strict priority, so that un-marked traffic in the low-latency queue cannot starve Classic traffic.

In Wi-Fi standards that support multi-Traffic-Identifier (multi-TID) frame aggregation (such as IEEE 802.11be), the link-layer subsystem SHOULD NOT bundle frames from the low-latency queue with traffic using longer transmission windows.


### Packet Classification

Where a local low-latency queue is used, the subsystem MUST steer uplink packets carrying the L4S identifier in the IP header to the low-latency queue, and SHOULD also support mapping based on the NQB DSCP value (45) {{RFC9956}} as an alternative or supplementary local classifier. Because DSCP markings are frequently bleached or remarked at network boundaries, ECN mapping remains the most reliable end-to-end classifier across mobile networks.

Where a device uses local host metadata (such as per-application or per-user priority markings) to steer traffic into a high-priority queue, classification based on network-signalled bearer rules, the L4S identifier, and NQB markings MUST take precedence so that low-latency packets from prioritised applications are not diverted into a queue-building queue.

Link-layer subsystems MUST NOT attempt to dynamically classify packets for a low-latency queue using heuristic traffic inference or Deep Packet Inspection (DPI). Apart from network-signalled bearer mapping rules ({{low-latency-queueing}}), classification MUST rely solely on explicit packet markings set by the application endpoints. This ensures compatibility with fully encrypted payloads and aligns with the end-to-end principle and permissionless innovation, as discussed in the ISP deployment observations in {{I-D.livingood-low-latency-deployment}} (which also contains details on Wi-Fi link-layer queueing considerations).

### Defence Against Misbehaving Traffic {#defense-against-misbehaving-traffic}

Applications may erroneously or maliciously mark queue-building or unresponsive traffic with the L4S identifier or NQB DSCP without implementing responsive congestion control. Because the device's low-latency queue does not apply `CE` marking ({{uplink-ce-marking}}), such traffic can overrun the shallow low-latency buffer and induce queueing delay or packet loss for well-behaved low-latency flows sharing the queue.

To bound queueing delay and prevent starvation of Classic traffic, the low-latency queue MUST enforce a shallow buffer limit (dropping or redirecting excess packets upon overflow) and a bounded scheduling weight as described in {{low-latency-queueing}} (Section 5.1 of {{RFC9956}}). In addition, link-layer subsystems (or the host operating system) SHOULD implement traffic protection (Section 5.2 of {{RFC9956}}) to detect queue-building flows in the low-latency queue and reclassify (demote) them to a queue-building queue. Where maintaining per-flow state in modem or Wi-Fi subsystem is impractical due to hardware limitations, or simply ineffective (e.g. across encrypted VPN tunnels as described in Section 4.1 of {{RFC9332}}), implementations MAY rely on the fallback mechanisms described in {{no-ce-marking-path}}.

### Uplink Transparency and Bleach Prevention

The link layer MUST NOT modify the ECN field, DSCP values, or AccECN TCP options (172 and 174) on outbound traffic.

## Behaviour on a Path That Never Marks CE {#no-ce-marking-path}

Sender on a mobile device might traverse a paths where no nodes leading to and including the node that is the transmission bottleneck apply `CE` markings.

Where `ECT(1)` traffic shares a standard queue with Classic traffic (such as on a default cellular radio bearer without a local low-latency queue), compliant L4S congestion controllers fall back to Classic congestion control behaviour upon packet loss (Section 4.3 of {{RFC9331}}), achieving throughput and fairness comparable to Classic flows without further intervention.

However, where the device steers `ECT(1)` packets into a shallow local low-latency queue ({{low-latency-queueing}}) and no downstream node marks `CE`, a capacity-seeking L4S flow will build a queue locally and suffer premature shallow-buffer drops, degrading both its own throughput and the latency of other flows sharing that queue. To mitigate this:

1. Where per-flow traffic protection is supported (Section 5.2 of {{RFC9956}}), the link-layer subsystem (or host operating system) SHOULD selectively demote queue-building flows from the low-latency queue to a Classic queue. Similarly, when a sending transport on the host operating system observes repeated loss episodes on a connection while receiving zero `CE` marks, it SHOULD fall back to Classic congestion control and stop marking outgoing packets as `ECT(1)` for that flow (Section 4.3 of {{RFC9331}}), which steers subsequent packets into the Classic queue without requiring per-flow state in link-layer hardware.
2. If shallow low-latency queue overflows persist across multiple flows, or if per-flow isolation is not possible, the device SHOULD disable `ECT(1)` marking on a per-destination or per-network basis. To prevent overly eager disabling of L4S in networks experiencing temporary issues, this decision should be ephemeral (instead of cached as in {{per-network-detection}}).

# Security Considerations

L4S introduces potential abuse vectors where applications mark queue-building traffic as low-latency. This risk is amplified if networks without low-latency queues instead prioritise `ECT(1)`-marked traffic using alternative heuristics (such as bandwidth allocation multipliers) without an active end-to-end congestion-marking feedback loop, incentivising endpoints to mark `ECT(1)` without implementing L4S congestion control. Regardless of network behaviour, host stacks and applications MUST NOT mark traffic as `ECT(1)` unless using compliant L4S congestion control. As described in {{defense-against-misbehaving-traffic}}, the link-layer subsystem MUST implement queueing and scheduling safeguards to defend against starvation and unbounded queueing delay.

# IANA Considerations

This document has no IANA actions.

--- back
