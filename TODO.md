# AI suggestions for topics to further explore

This document contains a list of suggested improvements, RFC alignments, and technical considerations to explore for future revisions of the mobile L4S deployment BCP.

## 1. Host OS Requirements

### Per-Packet ECN Control for Multiplexed Sockets
*   **Topic**: Sockets that multiplex multiple traffic types (e.g., WebRTC multiplexing RTP media, RTCP control, and SCTP data channels on a single UDP socket) cannot use simple socket-wide `setsockopt(IP_TOS)` because some flows (like SCTP) are queue-building and must not be marked `ECT(1)`.
*   **Suggestion**: Recommend that host OSes expose per-packet ECN marking APIs (such as `sendmsg()` with `IP_TOS` / `IPV6_TCLASS` control messages in `cmsg` data) so applications can selectively mark L4S-compatible packets on shared sockets.

### RFC 8888 Alignment for UDP Interactive Media
*   **Topic**: Standardizing ECN feedback for RTP-based real-time interactive media streams.
*   **Suggestion**: Align UDP requirements with **RFC 8888** ("RTP Control Protocol (RTCP) Feedback for Congestion Control on Interactive Real-Time Media"), recommending that interactive media applications use the RFC 8888 feedback format to echo ECN CE markings back to the sender.

---

## 2. Link-Layer Subsystems Requirements (Modem & Wi-Fi)

### Dynamic Time-Based Queue Sizing
*   **Topic**: Static byte-limit queues (e.g., a hard 16kB queue size) do not scale well on mobile networks where throughput fluctuates from <1 Mbps on weak LTE to 1+ Gbps on 5G.
*   **Suggestion**: Explore recommending dynamic time-based queue sizing (sojourn time limits, e.g., sizing the queue to hold no more than 10-15 ms of data at the current transmission capacity) rather than a static byte/packet limit.

### Cellular Radio Bearer Configuration (RLC Mode)
*   **Topic**: Bypassing in-order link-layer delivery to reduce latency spikes.
*   **Suggestion**: Provide specific guidance recommending that cellular networks map L4S (`ECT(1)`) and NQB (`DSCP-45`) traffic to radio bearers configured with **RLC Unacknowledged Mode (UM)** or with in-order delivery disabled at the radio link control layer, rather than standard RLC Acknowledged Mode (AM).

### RFC 9332 Alignment (DualQ Coupled AQM)
*   **Topic**: Scheduling and coupling configurations for multi-queue schedulers.
*   **Suggestion**: Ensure that multi-queue scheduling and coupling algorithms in the modem or Wi-Fi driver align with the coupled marking and AQM parameters defined in **RFC 9332** to ensure fair resource sharing between L4S and Classic queues.

### RFC 8325 Alignment (DiffServ to 802.11 Wi-Fi Mapping)
*   **Topic**: Mapping DSCP-45 (NQB) to Wi-Fi User Priorities and Access Categories.
*   **Suggestion**: Align Wi-Fi classification with **RFC 8325**, suggesting that Wi-Fi access points and client drivers map NQB (`DSCP-45`) traffic to appropriate WMM Access Categories (typically `AC_VI` or `AC_BE` with optimized queue settings).

---

## 3. Middlebox & Carrier Requirements

### NQB Marking Demotion at Carrier Ingress Boundaries
*   **Topic**: Handling DSCP-45 traffic in carrier networks that do not support NQB queue protection.
*   **Suggestion**: Recommend that ingress nodes (UPFs/PGWs) that lack queue protection capabilities demote unrecognized `DSCP-45` traffic to `DSCP-0` (Best Effort) rather than dropping the packets, to maintain robust backward compatibility.
