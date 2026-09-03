---
title: "Best Practices for L4S implementation for Mobile Devices"
abbrev: "Mobile L4S"
category: bcp
docname: draft-sporeba-tsvwg-mobile-l4s-latest
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
  RFC3168:
  RFC8311:
  RFC9330:
  RFC9331:
  RFC9332:
  RFC9768:
  RFC9956:

informative:
  RFC8325:
  IEEE-802.11-2024:
    target: "https://doi.org/10.1109/IEEESTD.2025.10979691"
    title: "IEEE Standard for Information Technology--Telecommunications and Information Exchange between Systems - Local and Metropolitan Area Networks--Specific Requirements - Part 11: Wireless LAN Medium Access Control (MAC) and Physical Layer (PHY) Specifications"
    author:
      - org: "IEEE"
    date: 2025-04
    seriesinfo:
      IEEE: "Std 802.11-2024"
      DOI: "10.1109/IEEESTD.2025.10979691"
  TS38.211:
    target: "https://www.3gpp.org/dynareport/38211.htm"
    title: "NR; Physical channels and modulation"
    author:
      - org: "3GPP"
    seriesinfo:
      3GPP: "TS 38.211"
  TS36.211:
    target: "https://www.3gpp.org/dynareport/36211.htm"
    title: "Evolved Universal Terrestrial Radio Access (E-UTRA); Physical channels and modulation"
    author:
      - org: "3GPP"
    seriesinfo:
      3GPP: "TS 36.211"
  I-D.livingood-low-latency-deployment:
  I-D.ietf-tsvwg-udp-ecn:

...
--- abstract

This document documents best practices for deployment of Low Latency, Low Loss, and Scalable Throughput (L4S) in mobile devices. It defines the responsibilities of the host operating system, the link-layer (modem and WiFi) subsystems to ensure successful end-to-end low-latency communication.

--- middle

# Introduction

L4S (Low Latency, Low Loss, Scalable Throughput) {{RFC9330}} offers a framework to significantly reduce queuing delay and loss while maintaining consistent throughput. Mobile devices often have to react to quickly changing connectivity conditions and may be subject to variable throughput and connection quality. This can cause large variations in user-perceived latency and greater bufferbloat than in other devices. Deploying L4S in a mobile ecosystem requires co-operation across multiple layers: the network stack, the host operating system (OS), and link-layer drivers and firmware (e.g., Wi-Fi and cellular modem), and the network. This document outlines best current practices for each of these subsystems to achieve reliable, low-latency performance in the field.

## Conventions and Definitions

{::boilerplate bcp14-tagged}

# Host Operating System Requirements

The host operating system controls application-level network access and hosts the primary TCP and UDP transport stacks.

## Socket APIs for UDP

To enable UDP-based transport stacks (such as QUIC) to utilize L4S, the OS MUST provide APIs that allow applications to:

1. Set the ECN codepoint {{RFC9331}} on outgoing packets.
1. Read the ECN codepoints of incoming packets.

These capabilities MUST be exposed via standard socket APIs (e.g., `IP_TOS` and `IPV6_TCLASS` for setting, and `IP_RECVTOS` and `IPV6_RECVTCLASS` via ancillary data for reading) and MUST NOT be restricted by default security policies for standard application sockets. The APIs MUST allow reading the ECN codepoints on a per-packet basis. and MUST allow setting the ECN codepoints either on a per-socket or per-packet basis. The detailed explanation of configuring UDP sockets for ECN for common platforms is covered in {{I-D.ietf-tsvwg-udp-ecn}}. It should be noted that on some platforms setting ECN codepoints shares calls with DSCP codepoints, and applications might need to take steps to avoid overriding commands.

UDP-based transport stacks SHOULD mark L4S-capable traffic as ECT(1) and MUST NOT mark non-L4S traffic as ECT(1). Other NQB traffic marked with DSCP 45 codepoint as per {{RFC9956, Section 3.3}}. Such traffic is considered compatible with L4S and can share the low-latency queue, but should be marked with ECT(1) only to signify the responsiveness to ECN markings (i.e. Prague L4S requirements {{RFC9331}}. Link layers MUST respond to misbehaving stack as discussed in {#defense-against-misbehaving-traffic}.

## TCP support detection and bootstrapping

To allow L4S-capable senders (e.g., Internet servers) to take advantage of L4S, host operating systems and link-layers must negotiate ECN support and verify path integrity as described in Section 1.4 and Section 3 of {{RFC9768}}.

### Per-network detection and latency mitigation

If misconfigured networks or servers drop ECN negotiation packets, and the client does not retry without ECN negotiation, TCP connections will fail. Therefore, clients MUST implement retry strategies that disable ECN negotiation, as described in Section 3.1.4 and Section 3.2.3.2.2 of {{RFC9768}}. Note that a server replying with Not-ECT is does not require any additional retries and is not considered a failure.

Latency and loss are critical to mobile applications, and retry strategies dependent on retransmissions and timeouts can lead to a degraded user experience. To ensure that L4S can be safely enabled without degrading the user experience in the presence of networks or servers that drop ECN negotiations or Accurate ECN TCP options, the client SHOULD balance implementing the retry strategies in section 3 of {{RFC9768}} with mechanisms to reduce the latency impact of retransmissions. A possible strategy is to only attempt ECN negotiation on the first SYN to a new server. Additionally, the host MAY cache ECN negotiation timeouts on a per-host or per-IP-address, or other basis.

If a host is in a network that blocks all ECN negotiation regardless of destination, all TCP connections will suffer latency impact. This will degrade the user experience even if failures are cached, because any connection to a server that is not in the cache will suffer an additional round trip delay.

A host system that wants to be resilient to this MAY attempt a connectivity check to a known, L4S-supporting service. In case of check failure, the result can be used to turn off L4S negotiation attempts for a given network, represented by PLMN/APN (in carrier networks) or SSID/BSSID (in Wi-Fi networks).

When maintaining such lists, entries SHOULD be retired after a reasonable TTL (e.g. 7 days), and SHOULD prefer information from more frequently used destinations.

# Link-layer Subsystems Requirements

Link-layer subsystems, such as the cellular modem and Wi-Fi schedule and manage the link-layer transmission over the physical medium interface. These systems typically perform significant queueing on the transmit path, and sometimes on the receive path as well.

## Link-layer inbound packet reordering

Some link layers provide strong ordering guarantees for inbound packets by assigning a link-layer sequence number to each packet and buffering incoming packets so they can be presented to the host operating system in order. In particular, cellular networks often perform out-of-order packet delivery at the physical layer, but require the cellular modem to deliver received packets to the host operating system in order. When packets are received out of order on the air interface, the modem waits up to a network-configurable timeout to receive all previous packets. This causes out-of-order packets to be delayed until all previous packets have been received, and causes latency spikes when packets are lost due to transmission errors.

Delaying received packets increases latency, which is contrary to the low-latency goals of L4S. Also, by artificially introducing delays that were not imposed by the network, it reduces the accuracy of protocol rate estimation. The additional delays do not benefit protocol stacks which are equipped to handle out-of-order packets.

Link-layers MUST NOT buffer inbound L4S packets in a way that imposes measurable latency to the protocol stack.

{{RFC9331}} §4.3 specifies that loss detection SHOULD be resilient to packet reordering. However, to enable the requirement on the cellular modem behaviour described above, L4S-aware protocol stacks on mobile devices MUST be prepared to receive out-of-order packets.

## Multi-Queue Scheduling and Bounded Latency Queueing

Any link layer that supports L4S MUST provide a low-latency queue designated for L4S traffic and Non-Queue-Building traffic ({{RFC9956}} section 3.3). This queue MUST be bounded as described in {#uplink-aqm} and {#defense-against-misbehaving-traffic}.
Some link-layer systems already support high-priority and low-priority queues. These queues are typically not latency-bounded, and therefore cannot guarantee the low-latency benefits of L4S. If the link layer provides such queues, low-latency queue MUST be distinct from them. An example configuration might be:


*  **Low-Latency Queue:** For L4S and other Non-Queue-Building traffic.
*  **High-Priority Queue:** For high priority but potentially queue-building traffic.
*  **Low-Priority Queue:** For queue-building traffic (e.g., CUBIC/Reno) and bulk data.

Link layers MUST ensure that the L4S queue does not starve the other queues if offered L4S traffic is higher than available bandwidth. Such prioritisation SHOULD use a scheduling algorithm (e.g., Weighted Fair Queueing) and aim to minimize the queue buildup in the Low-Latency queue.

## Packet Classification

The link-layer MUST map uplink traffic to the low-latency queue based on ECN markings:

*  Packets carrying the `ECT(1)` or `CE` bits in the IP header MUST be steered to the low-latency queue.
*  The link-layer SHOULD also support mapping to the low-latency queue based on the Non-Queue-Building (NQB) DSCP value (45) {{RFC9956}} as an alternative or supplementary classifier. Because DSCP markings are frequently bleached at carrier interconnect boundaries, ECN mapping remains the most reliable end-to-end classifier for mobile networks.

Link-layer networks MUST NOT attempt to dynamically classify packets for the low-latency queue using heuristic traffic inference or Deep Packet Inspection (DPI). Classification MUST rely solely on the explicit packet markings set by the application endpoints. This ensures compatibility with fully encrypted payloads and aligns with the end-to-end principle and permissionless innovation, as discussed in the ISP deployment observations in {{I-D.livingood-low-latency-deployment}} (which also contains details on Wi-Fi link-layer queuing considerations).


## Uplink Active Queue Management (AQM) {#uplink-aqm}

The link-layer uplink buffer within mobile subsystems (such as cellular modems and Wi-Fi drivers) operates as a dynamic bottleneck subject to rapid capacity fluctuations driven by radio grant scheduling, carrier aggregation shifts, and physical layer channel fading. To help achieve consistently low latency under these volatile conditions without inducing throughput degradation, implementations MUST deploy a Dual-Queue Coupled AQM framework adhering strictly to the functional design specifications of {{RFC9332}}.

In alignment with {{RFC9332}} Section 2.4, all internal marking states, delay targets, and queue boundaries MUST be calculated using packet sojourn time (expected time-to-service) rather than raw bytes or static packet counts.

### Wi-Fi subsystem sojourn time recommendations

In a shared wireless medium such as Wi-Fi, transmission requires contention-based spectrum acquisition and relies on frame aggregation to achieve channel efficiency. Often a buffer once scheduled for transmission cannot be expanded, resulting in traffic bursts and a potential delay for subsequent packets. To prevent erroneously signaling congestion during normal frame aggregation and transmission bursts, Wi-Fi implementations SHOULD configure the sojourn time step threshold to longer than the maximum burst time. For example, in Wi-Fi 7 (802.11be) Access Category Best Effort (AC_BE), the maximum A-MPDU transmission is 4 MB, transmitted over a maximum airtime of approx. 5.5 ms. Therefore a reasonable sojourn time threshold to use would be 6 ms.

Wi-Fi subsystems SHOULD utilize MAC-layer QoS mechanisms to prioritize L4S traffic and reduce channel access delay. Default access categories (such as AC_BE) might experience a worst-case channel access wait exceeding the sojourn time defined by burst traffic protection.

For example, for Wi-Fi 7 (802.11be) AC_BE the maximum uninterrupted backoff countdown is approx. 9.2 ms ({{IEEE-802.11-2024}} Table 9-194). For Access Category Video (AC_VI) this time is only 0.135 ms. The standard also caps the transmission opportunity time (TXOPLimit) for AC_VI to typically 3 ms.

While it might sound attractive to raise the sojourn time threshold of AC_BE traffic to accommodate the total channel access delay experienced under heavy contention (for example to 30 ms), doing so would cause L4S congestion controls to maintain an unnecessarily large buffer, comparable with other non-ECN-responsive congestion controls.

Therefore, Wi-Fi subsystems SHOULD assign ECT(1) marked traffic to an Access Category more appropriate for low-latency traffic (such as Video - AC_VI). This is analogous to the DSCP-to-UP mapping described in {{RFC8325, Section 4.2}}. In these cases, the sojourn time might no longer be limited by A-MPDU, and implementations SHOULD use a lower sojourn time threshold derived from TXOPLimit (for example, a value of 4 ms for AC_VI).

In standards that support multi-TID frame aggregation (such as IEEE 802.11be), different access categories can be mixed into a single A-MPDU aggregate. In these scenarios the link-layer subsystem SHOULD NOT bundle latency-sensitive L4S frames with traffic using longer transmission windows.

### Cellular subsystem sojourn time recommendations

Unlike the contention-based channel access of Wi-Fi, cellular networks utilize centralized, grant-based uplink scheduling managed by the base station.

During sustained transmission, the base station allocates uplink resources based on buffer status reports in ongoing traffic. Without contention backoff, the queue drains on each radio scheduling interval. Cellular modem implementations SHOULD configure the native L4S sojourn time step threshold based on the radio scheduling interval and physical-layer retransmission turnaround. For example, a threshold of 2 ms is appropriate for 5G networks {{TS38.211}}, whereas 4 ms is appropriate for 4G networks {{TS36.211}}.

When a flow transitions from an idle buffer to an active state, the device must first request uplink transmission resources from the network. This initial scheduling turnaround can introduce a delay of approximately 3 ms to 6 ms. To prevent this initial delay from triggering spurious CE marks on the first packet of a burst or connection, implementations SHOULD apply an initial burst tolerance.

Where the cellular network provides pre-allocated uplink transmission resources (such as semi-persistent scheduling or configured grants), the initial scheduling turnaround is eliminated, allowing implementations to use the lower steady-state threshold directly.

Additionally, cellular link layers SHOULD minimize reordering delays on the radio interface. Where configurable, data bearers carrying L4S traffic SHOULD avoid link-layer in-order delivery enforcement (for example, utilizing unacknowledged mode or enabling link-layer out-of-order delivery). This ensures that physical layer retransmissions do not cause head-of-line blocking stalls in the modem buffer.



## Defense Against Misbehaving Traffic (Queue Protection) {#defense-against-misbehaving-traffic}

Applications may incorrectly or maliciously mark non-responsive traffic as NQB or `ECT(1)` without implementing a scalable, responsive L4S congestion control algorithm. Because a standard dual-queue coupled framework lacks per-flow processing, a single non-responsive L4S stream will inflate the coupled congestion metrics, driving up CE marking across the entire Low-Latency queue and increasing drops on the Classic queue side, unfairly penalizing well-behaved applications.

To defend the Low-Latency queue while remaining fully compliant with the normative boundaries of {{RFC9332}}, mobile link-layer subsystems MUST enforce the following safeguards:

* **Layered Congestion and Drop Progression:** Under standard operating conditions, congestion signaling for ECN-capable packets within the Low-Latency queue MUST rely entirely on CE marking. In accordance with the overload mandates of {{RFC9332}} Sections 2.5.1.1 and 4.2.3, if sustained saturation is detected (i.e., marking capacity is exhausted at 100% ECN signaling or the queue delay persistently exceeds operational bounds), the link-layer MUST introduce classic packet drops to both types of traffic to defend link integrity. Implementations SHOULD follow a strict architectural fallback progression: PI2 Gradual Marking -> Step AQM Burst Marking -> Overload Drops -> Tail-Drop.
* **Buffer Limit Metrics:** The link-layer MUST enforce a strict ceiling on the maximum physical capacity of the Low-Latency queue. To prevent premature packet drops when handling heavily aggregated payloads, this absolute limit SHOULD be defined in terms of packet count (with a recommended default limit of 10,000 packets) rather than raw bytes, operating strictly as a final tail-drop mechanism at physical hardware capacity exhaustion.
* **Flow Isolation Architecture:** As noted in {{RFC9332}} Section 4.2.2, scheduling weights alone cannot resolve long-term overload across a shared buffer. For mobile device environments requiring robust traffic defense against rogue applications, the link-layer MAY layer a per-flow queueing discipline or flow-fairness mechanism underneath the macro coupled dual-queue architecture. This design architecture isolates non-responsive streams into distinct internal queueing slots, shielding well-behaved applications from the coupled penalty while fully preserving the macro coupled framework requirements.

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

Network deployments that don't support low-latency queues, and instead prioritise ECN marked traffic based on alternative heuristics (such as bandwidth allocation multipliers) are not an L4S deployment. The risk of such alternative mechanisms is incentivising marking `ECT(1)` codepoints in flows to be allocated into prioritised queues, without implementing full L4S congestion control as described in {#defense-against-misbehaving-traffic}. The convergence point of such behaviour would be a prevalence of `ECT(1)` marked traffic that does not respond to `CE` markings and congestion in the incorrectly prioritised queues.

# Security Considerations

L4S introduces potential abuse vectors where applications mark queue-building traffic as low-latency. As described in {#defense-against-misbehaving-traffic}, the link-layer subsystem MUST deploy queue protection mechanisms to defend the low-latency queue from starvation and latency degradation.

# IANA Considerations

This document has no IANA actions.

--- back
