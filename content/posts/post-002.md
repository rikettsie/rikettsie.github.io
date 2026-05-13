---
title: "The Signal protocol: X3DH vs PQXDH, and the Double Ratchet"
date: 2023-10-20
draft: false
tags: ["cryptography", "signal", "post-quantum"]
summary: "How Signal achieves forward secrecy, break-in recovery, and deniability — from X3DH to PQXDH and the Double Ratchet."
---

## First of all: is Signal p2p?

Signal is not peer-to-peer at the transport layer. Messages are routed
through Signal's central servers, which hold prekeys, queue messages for
offline recipients. They are a required intermediary, i.e. you cannot use
Signal without Signal's infrastructure.

At the *cryptographic* layer, however, the server is a blind relay. It
never sees plaintext, and with sealed sender it does not learn who is
talking to whom. The trust model is end-to-end: the server is in the path
but not in the circle of trust.

Thus, in contrast with truly p2p messaging systems like Matrix, Signal trades decentralisation for simplicity with a tighter, heavier, security model.

The rest of this post is about how this security model is built; it has been
very interesting to me to learn all these bits.

## Signal's security model

Signal's security goal is strict, and it targets three distinct properties.

**Forward secrecy**: even if a server is compromised today, past messages
stay private, i.e. old message keys are deleted and cannot be reconstructed.

**Break-in recovery**: even if a client is compromised and current keys
are stolen, the attacker loses access again after the next ratchet step.

**Deniability**: Signal produces no unforgeable proof of authorship. Even
if someone records your messages, they cannot prove to a third party that
you wrote them — the other party could have forged any message, unlike
with PGP signatures which bind a message irrevocably to a key.

Reaching this goal requires layering three cryptographic ideas.

## 1. Asynchronous key agreement

### 1a. X3DH — Extended Triple Diffie-Hellman

The **X3DH** protocol lets one peer (Alice) start encrypting
messages to the other (Bob) even when this one is offline.

Bob registers with the server three kinds of keys:
- **IK_B** — long-term identity key
- **SPK_B** — medium-term signed prekey (rotated weekly), signed by IK_B
- **OPK_B** — a batch of one-time prekeys (used once, ever)

Alice generates an ephemeral key **EK_A** and computes four DH operations:

```
DH1 = DH(IK_A, SPK_B)
DH2 = DH(EK_A, IK_B)
DH3 = DH(EK_A, SPK_B)
DH4 = DH(EK_A, OPK_B)   # omitted if no OPK available
```

The shared secret is `KDF(DH1 || DH2 || DH3 || DH4)`. Each DH covers a
different security property: IK-to-IK for authentication, EK-to-IK for
forward secrecy, EK-to-SPK for binding to Bob's current key, EK-to-OPK
for one-time deniability.

### 1b. PQXDH — X3DH's post-quantum successor

X3DH was deprecated by Signal this September 2023 and replaced with **PQXDH**
(Post-Quantum Extended Diffie-Hellman) which was released in the client application version 6.35.
The reason of deprecation is the *harvest now, decrypt later* threat:
an adversary can record encrypted traffic today and store it until
a large-scale quantum computer becomes available. Shor's algorithm would then
break the elliptic curve Diffie-Hellman at the heart of X3DH, retroactively
exposing every past session.

PQXDH addresses this by adding a post-quantum KEM
(standing for *Key Encapsulation Mechanism*), the **CRYSTALS-Kyber**,
alongside the existing X25519 exchange. Bob publishes an additional Kyber
prekey; Alice encapsulates a secret under it and mixes
the result into the shared secret alongside the
four classical Diffie-Hellman (DH) outputs. An attacker must break *both* the classical
and the post-quantum component to compromise the session, it's hard.

All of X3DH's goals are preserved: asynchronous key agreement, forward
secrecy, deniability, and the same prekey registration flow. The Double
Ratchet that follows is unchanged.

## 2. The Double Ratchet — per-message key evolution

Once X3DH establishes a root key, the Double Ratchet takes over for the
conversation. It combines two ratchets:

**Symmetric-key ratchet (KDF chain)**: each message derives the next
message key by hashing the current chain key. Deleting used keys gives
*forward secrecy* — a stolen device cannot decrypt past messages.

**Diffie-Hellman ratchet**: every time the other party replies, both sides
perform a new DH exchange and inject the result into the root chain. This
gives *break-in recovery* — if an attacker steals your current keys, they
lose access as soon as the next DH ratchet step occurs.

## 3. Sealed sender

By default, the Signal server learns who is messaging whom.
Sealed sender feature (introduced in 2018) wraps the sender's identity inside
the encrypted payload so the server sees only the recipient.
Combined with the server's lack of metadata logging, this feature challenges traffic analyzers.

## Design rationale

The three layers (key agreement, double ratchet, sealed sender) close
a specific gap, targeting distinct threats:
- PQXDH: bootstrap without a live session
- Double Ratchet: key compromise containment over time
- Sealed sender: minimizes peer metadata knoledge by the infrastructure

The formal security model was verified by Cohn-Gordon et al. (2016), who
proved the Double Ratchet achieves "message secrecy" and "break-in
recovery" under standard assumptions.

## References

- [libsignal — rust/protocol/src/](https://github.com/signalapp/libsignal/tree/main/rust/protocol/src) — official implementation; `ratchet/` and `keys/` are the core
- [Signal-Android](https://github.com/signalapp/Signal-Android) — `app/src/main/java/org/thoughtcrime/securesms/crypto/`
- [python-doubleratchet](https://github.com/Syndace/python-doubleratchet) — pedagogical standalone Double Ratchet

- Marlinspike, M., & Perrin, T. (2016). **The X3DH key agreement protocol**. Signal Foundation. https://signal.org/docs/specifications/x3dh/
- Perrin, T. (2023). **The PQXDH key agreement protocol**. Signal Foundation. https://signal.org/docs/specifications/pqxdh/
- Marlinspike, M., & Perrin, T. (2016). **The Double Ratchet algorithm**. Signal Foundation. https://signal.org/docs/specifications/doubleratchet/
- Cohn-Gordon, K., Cremers, C., Dowling, B., Garratt, L., & Stebila, D. (2016). **A formal security analysis of the Signal messaging protocol**. *IEEE European Symposium on Security and Privacy (EuroS&P 2017)*. (ePrint 2016/1013)
- Unger, N., et al. (2015). **SoK: Secure messaging**. *IEEE Symposium on Security and Privacy (S&P 2015)*.
