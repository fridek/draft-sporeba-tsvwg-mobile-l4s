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

...
--- abstract

This document describes best practices for deployment of Low Latency, Low Loss, and Scalable Throughput (L4S) in mobile devices. It defines the responsibilities of the host operating system and the link-layer (modem and Wi-Fi) subsystems to ensure successful end-to-end low-latency communication.

--- middle

# Introduction

L4S (Low Latency, Low Loss, Scalable Throughput) {{RFC9330}} offers a framework to significantly reduce queueing delay and loss while maintaining consistent throughput. Mobile devices often have to react to quickly changing connectivity conditions and may be subject to variable throughput and connection quality. This can cause large variations in user-perceived latency and greater bufferbloat than in other devices. Deploying L4S in a mobile ecosystem requires co-operation across multiple layers: the network stack, the host operating system (OS), link-layer drivers and firmware (e.g., Wi-Fi and cellular modem), and the network. This document outlines best current practices for the mobile device subsystems to achieve reliable, low-latency performance in the field.

## Conventions and Definitions

{::boilerplate bcp14-tagged}

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

Link-layer subsystems, such as the cellular modem and Wi-Fi, schedule and manage the link-layer transmission over the physical medium interface. These systems typically perform significant queueing on the transmit path, and sometimes on the receive path as well.

## Link-Layer Inbound Packet Reordering

Some link layers provide strong ordering guarantees for inbound packets by assigning a link-layer sequence number to each packet and buffering incoming packets so they can be presented to the host operating system in order. In particular, cellular networks often perform out-of-order packet delivery at the physical layer, but require the cellular modem to deliver received packets to the host operating system in order. When packets are received out of order on the air interface, the modem waits up to a network-configurable timeout to receive all previous packets. This causes out-of-order packets to be delayed until all previous packets have been received, and causes latency spikes when packets are lost due to transmission errors.

Delaying received packets increases latency, which is contrary to the low-latency goals of L4S. Also, by artificially introducing delays that were not imposed by the network, it reduces the accuracy of protocol rate estimation. The additional delays do not benefit protocol stacks which are equipped to handle out-of-order packets.

Link-layer subsystems MUST NOT buffer outbound and inbound `ECT(0)`, `ECT(1)` and `CE` marked packets in a way that imposes measurable latency to the protocol stack. Correspondingly, cellular modems SHOULD ensure that `ECT(0)`, `ECT(1)` and `CE` marked packets can be delivered out-of-order rather than delay to deliver them in-order. This ensures that air-interface retransmissions do not cause head-of-line blocking stalls.

Section 4.3 of {{RFC9331}} specifies that loss detection SHOULD be resilient to packet reordering. However, to enable the requirement on the cellular modem behaviour described above, L4S-aware protocol stacks on mobile devices MUST be prepared to receive out-of-order packets.

TODO: `ECT(0)`, `ECT(1)` and `CE` and `ECT(1)` or `CE` need to be collectively defined (e.g. using the definitions defined in RFC9332).

## Multi-Queue Scheduling and Bounded Latency Queueing

Any link layer that supports L4S MUST provide a low-latency queue designated for L4S traffic () and Non-Queue-Building traffic (as defined by Section 3.3 of {{RFC9956}}). This queue MUST be bounded as described in {{uplink-aqm}} and {{defense-against-misbehaving-traffic}}.

Some link-layer systems already support high-priority and low-priority queues. These queues are typically not latency-bounded, and therefore cannot guarantee the low-latency benefits of L4S. If the link layer provides such queues, the low-latency queue MUST be distinct from them. An example configuration might be:

*  **Low-Latency Queue:** For L4S and other Non-Queue-Building traffic.
*  **High-Priority Queue:** For high-priority but potentially queue-building traffic.
*  **Low-Priority Queue:** For queue-building traffic (e.g., CUBIC/Reno) and bulk data.

Link layers MUST ensure that the L4S queue does not starve the other queues if offered L4S traffic is higher than available bandwidth. Such prioritisation SHOULD use a scheduling algorithm (e.g., using the methods described in Section 4.2.2 of {{RFC9332}}) and aim to minimise the queue buildup in the low-latency queue.

## Packet Classification

The link layer MUST map uplink traffic to the low-latency queue based on ECN markings:

*  Packets carrying the `ECT(1)` or `CE` codepoints in the IP header MUST be steered to the low-latency queue.
*  The link layer SHOULD also support mapping to the low-latency queue based on the NQB DSCP value (45) {{RFC9956}} as an alternative or supplementary classifier for local uplink queueing. However, because DSCP markings are frequently bleached or remarked at network boundaries, ECN mapping remains the most reliable end-to-end classifier across mobile networks.

Link-layer networks MUST NOT attempt to dynamically classify packets for the low-latency queue using heuristic traffic inference or Deep Packet Inspection (DPI). Classification MUST rely solely on the explicit packet markings set by the application endpoints. This ensures compatibility with fully encrypted payloads and aligns with the end-to-end principle and permissionless innovation, as discussed in the ISP deployment observations in {{I-D.livingood-low-latency-deployment}} (which also contains details on Wi-Fi link-layer queueing considerations).

## Uplink Active Queue Management (AQM) {#uplink-aqm}

The link-layer uplink buffer within mobile subsystems (such as cellular modems and Wi-Fi drivers) operates as a dynamic bottleneck subject to rapid capacity fluctuations driven by radio grant scheduling, carrier aggregation shifts, and physical layer channel fading. To help achieve consistently low latency under these volatile conditions without inducing throughput degradation, implementations MUST deploy a Dual-Queue Coupled AQM framework adhering strictly to the functional design specifications of {{RFC9332}}.

As mentioned in Notes a. to Appendix A.1 of {{RFC9332}}, the drain rate of the queue can vary if it is scheduled relative to other queues or if it accommodates fluctuations in a wireless medium. To auto-adjust to changes in drain rate, the queue needs to be measured in time (e.g., packet sojourn time), not bytes or packets. This approach naturally accommodates large aggregated frame structures (e.g., up to 64kB aggregated packets passed via GSO) without triggering premature congestion signals.

TODO: RFC9332 section 2.5.1 specifies all the relevant low-latency queue behaviours. This needs to be quoted early (possibly by including here and moving this section above Multi-Queue Scheduling and Bounded Latency Queueing. We need to reconsile how DualPI2 works in the presence of pre-exising high-priority and low-priority queues (i.e. when there are three queues, one NQB and two queue-building).


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

## Defence Against Misbehaving Traffic (Queue Protection) {#defense-against-misbehaving-traffic}

Applications may incorrectly or maliciously mark non-responsive traffic as NQB or `ECT(1)` without implementing a scalable, responsive L4S congestion control algorithm. Because a standard dual-queue coupled framework lacks per-flow processing, a single non-responsive L4S stream will inflate the coupled congestion metrics, driving up `CE` marking across the entire low-latency queue and increasing drops on the Classic queue side, unfairly penalising well-behaved applications.

To defend the low-latency queue while remaining fully compliant with the normative boundaries of {{RFC9332}}, the following safeguards apply to mobile link-layer subsystems:

* **Overload Drop Fallback:** Under standard operating conditions, congestion signalling for `ECT(1)` packets within the low-latency queue MUST rely entirely on `CE` marking. Under abnormal conditions (i.e., 100% of the packets in the queue are `CE` marked or the queue delay persistently exceeds operational bounds), the link layer MUST introduce packet drops as specified in Sections 2.5.1 and 4.2.3 of {{RFC9332}}.
* **Buffer Limit Metrics:** In order to prevent misbehaving traffic from filling up the low-latency queue, the link layer MUST enforce a strict ceiling on the maximum physical capacity. The queue size should be reasonably small, to drop misbehaving packets early and trigger congestion control backoff. Conversly, to prevent premature packet drops when handling heavily aggregated payloads, this absolute limit SHOULD be defined in terms of packet count rather than raw bytes, and be appropriate for the transport medium.
* **Flow Isolation Architecture:** As noted in Section 4.2 of {{RFC9332}}, scheduling weights alone cannot resolve long-term overload across a shared buffer. For mobile device environments requiring robust traffic defence against rogue applications, the link layer MAY layer a per-flow queueing discipline or flow-fairness mechanism underneath the macro coupled dual-queue architecture. This architecture isolates non-responsive streams into distinct internal queueing slots, shielding well-behaved applications from the coupled penalty while fully preserving the macro coupled framework requirements.

TODO: This section needs to be revisited and aligned with the rest of the text. The overload drop fallback needs to be expanded on (what specific problem does it solve). Buffer limit metrics are potentially redundant in presence of other text, and because the finite nature of the queue is unavoidable. Flow isolation is difficult to implement in practice and we need to consider its necessity.

## Transparency and Bleach Prevention

The link layer MUST NOT modify the ECN field, DSCP values, or AccECN TCP options (172 and 174) on transit traffic, except for performing standard `CE` marking in the event of the queue buildup.

# Security Considerations

L4S introduces potential abuse vectors where applications mark queue-building traffic as low-latency. This risk is amplified if networks without low-latency queues instead prioritise `ECT(1)`-marked traffic using alternative heuristics (such as bandwidth allocation multipliers) without an active end-to-end congestion-marking feedback loop, incentivising endpoints to mark `ECT(1)` without implementing L4S congestion control. Regardless of network behaviour, host stacks and applications MUST NOT mark traffic as `ECT(1)` unless using compliant L4S congestion control. As described in {{defense-against-misbehaving-traffic}}, the link-layer subsystem MUST deploy queue protection mechanisms to defend the low-latency queue from starvation and latency degradation.

# IANA Considerations

This document has no IANA actions.

--- back
