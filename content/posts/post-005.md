---
title: "NAT traversal and hole punching: from UDP to QUIC with iroh"
date: 2025-11-08
draft: false
tags: ["hole-punching", "iroh", "NAT-traversal", "QUIC"]
summary: "How hole punching works, why TCP and plain UDP fall short, and how QUIC and iroh make direct P2P connections reliable."
---

Most devices on the internet sit behind a NAT: the router assigns them a
private address and rewrites packet headers on the way out. Two peers
behind separate NATs cannot reach each other directly, neither knows the
other's public address, and neither NAT will forward unsolicited packets.

Hole punching is the technique that makes direct connections possible
despite this.

## How NAT works (briefly)

When a device behind NAT sends a packet out, the NAT creates a *mapping*:
`(private_ip:private_port) -> (public_ip:public_port)`.
Packets returning to that public endpoint within the mapping lifetime
are forwarded inward.
The NAT *does not* forward packets arriving at a public port that has no
existing mapping, i.e. that have not previously been triggered.

## The hole-punching sequence

Both peers connect to a *relay server* (also called *signalling server*)
with a known public address. The relay can see each peer's public IP and port
(that's usually called the "reflexive address") and forwards this information
to both peers at each side. Now:

1. **Peer A** sends a packet to peer B's reflexive address.
   B's NAT drops it (no mapping exists), **but** A's NAT *creates a mapping*, i.e. **an "OK, GO" at A's side of the pipe!**, for future incoming traffic from B's public address.

2. **Peer B** simultaneously (or almost) sends a packet to A's reflexive address.
   A's NAT at that point **leverage** the previously established mapping for B's address and forwards the packet.
   B's NAT creates its own mapping for A.


Both sides of the pipe, open at *roughly* the same time. **The hole is made!** Subsequent packets flow through
without the relay.

## Limitations of TCP and plain UDP

Hole punching over **TCP** is theoretically possible via simultaneous open —
both peers send SYN packets to each other at exactly the same time, so each
NAT creates a mapping before the other's SYN arrives. In practice this is
brittle on two counts. First, some NATs track TCP state and actively block
an incoming SYN that arrives at a port with only an outgoing SYN and no
completed handshake — treating it as a potential SYN flood rather than a
legitimate simultaneous open. Second, symmetric NATs assign a different
external port per destination, so the reflexive address learned via the
relay is useless for the direct SYN. The result is that TCP hole punching
success rates vary too widely across NAT implementations to rely on.

**Plain UDP** hole punching works more reliably, but leaves everything else
to the application. NAT mappings for UDP typically expire after 30–300
seconds of inactivity, so the application must send keepalives or risk the
hole closing. More critically, if the local IP or port changes — a mobile
device switching from Wi-Fi to cellular, for instance — the mapping is gone
and the whole punching sequence must restart from scratch. There is also no
built-in encryption, ordering, or retransmission: the application owns all
of that.

## Why QUIC helps

UDP hole punching existed long before QUIC, but QUIC makes the post-connection
cleaner. QUIC connections are identified by a **connection ID** embedded in
the packet, not by the 4-tuple `(src_ip, src_port, dst_ip, dst_port)`.
This means:

- If a NAT remaps the port after connection (common on mobile), the QUIC
  connection can survive via a **connection migration** leveraging this ID.
- You can attempt multiple paths simultaneously and promote whichever
  succeeds first without restarting the handshake.

## iroh's approach

iroh combines three layers:

1. **Relay servers**: they are always-on fallback, primarily used to exchange reflexive addresses, but in extreme cases to route data through as well.
2. **Direct path upgrade**: iroh continuously attempts UDP hole punching;
   once a direct path is confirmed it is used for all data.
3. **Path monitoring**: if the direct path goes silent or stale (NAT mapping
   expired), iroh falls back to the relay and reattempts punching.

The result is a connection that starts instantly via relay and silently
upgrades to direct, with transparent fallback, without the application
layer needing to know.

## Symmetric NAT: the hard case

Symmetric NAT assigns a *different* public port for each destination.
The reflexive address seen by the relay is not the address a direct
packet from the other side would arrive on.
iroh tries to handle this with port prediction with euristics, but there
is no general guarantee of direct connectivity through symmetric NAT.

## Code refs

- [iroh-net — src/magicsock/](https://github.com/n0-computer/iroh/tree/main/iroh-net/src) — hole punching and path management
- [iroh relay server](https://github.com/n0-computer/iroh/tree/main/iroh-relay) — the relay used to exchange reflexive addresses
- [pion/stun](https://github.com/pion/stun) — STUN implementation in Go

## Bibliography

- Ford, B., Srisuresh, P., & Kegel, D. (2005). **Peer-to-peer communication across network address translators**. *Proceedings of the USENIX Annual Technical Conference (USENIX ATC '05)*.
- Iyengar, J., & Thomson, M. (2021). **QUIC: A UDP-based multiplexed and secure transport**. RFC 9000. IETF.
- Rosenberg, J., et al. (2008). **Session Traversal Utilities for NAT (STUN)**. RFC 5389. IETF. (updated by RFC 8489, 2020)
- Rosenberg, J., et al. (2010). **Traversal Using Relays around NAT (TURN)**. RFC 5766. IETF.
- Seemann, M., & Huitema, C. (2024). **Implementing NAT hole punching with QUIC**. arXiv:2408.01791. https://arxiv.org/pdf/2408.01791
