---
title: "iroh-rings: ring-based access control for P2P resources"
date: 2026-05-15
draft: false
tags: ["rust", "iroh", "p2p", "access-control", "iroh-rings"]
summary: "How I designed a ring-based access control library on top of iroh — and why the Registry trait is the most important design decision in it."
---

After spending time with iroh's transport layer — hole punching, QUIC connections,
NAT traversal — I wanted to build something on top of it. The connectivity problem
was solved. The next question was: *who gets to access what?*

In a centralised system this is trivial: you have a server, the server checks credentials,
done. In a P2P system there is no server to ask. Every node needs to decide for itself
whether to serve a resource to a remote peer. I needed an access control model that
was simple to reason about, did not require a central authority, and could plug into
iroh's existing identity model cleanly.

That library is [iroh-rings](https://crates.io/crates/iroh-rings).

## The ring model

The central concept is a **ring**: a named group of peers. A resource is associated
with one or more rings. A peer can access a resource if and only if it is a member
of at least one of those rings.

That is the entire model. No roles, no capabilities, no ACL lists. Just named groups
and membership.

There is one built-in ring: the **open ring** (`OPEN_RING_NAME`). Resources associated
with it are accessible to any peer, with no membership check. It is the public ring —
useful for resources you want to share with the world without managing a list of peers.

The mental model maps naturally onto real access patterns:

- A file shared with your team -> associate it with a `team` ring containing your
  colleagues' endpoint IDs.
- A file shared publicly -> associate it with the open ring.
- A file shared with a specific peer -> create a ring with just that peer in it.

## The Registry trait

The `Registry` trait is the core abstraction. It manages three things: rings, their
peer membership, and the association between resources and rings. Any struct that
implements it, can back the system — the access control logic doesn't changes.

```rust
pub trait Registry {
    fn create_ring(&self, ring_name: &str) -> Result<()>;

    fn add_peer_to_ring(
        &self,
        ring_name: &str,
        peer: EndpointId,
        nickname: Option<&str>,
    ) -> Result<()>;

    fn add_ring_to_resource<ResId: ResourceId>(
        &self,
        resource_id: ResId,
        ring_name: &str,
    ) -> Result<()>;

    fn is_allowed<ResId: ResourceId>(
        &self,
        peer: &EndpointId,
        resource_id: &ResId,
    ) -> Result<bool>;

    // ... list_rings, list_ring_peers, remove_peer_from_ring, list_resource_rings
}
```

A few things worth noting in this signature. All methods take `&self`, even the
ones that write — `create_ring`, `add_peer_to_ring`, `add_ring_to_resource` all
mutate state, yet none takes `&mut self`.

This is a deliberate choice: requiring
`&mut self` would mean callers need exclusive access to the registry for every
write, which becomes awkward when the registry is shared across threads or owned
behind an `Arc`. By using `&self`, the trait leaves the synchronization strategy
to the implementor. `RedbRegistry`, for example, wraps a redb `Database` which
opens its own write transactions internally — no external `&mut` needed. An
implementor that wraps a `HashMap` would use a `Mutex<HashMap>` or `RwLock` instead.
The trait does not prescribe which; it only says "you need shared access, figure
out the rest".

Methods that touch resources are generic
over `ResId: ResourceId` rather than using an associated type, so you can use
different resource ID types with the same registry instance. The authorization rule
is: a peer is allowed if it belongs to at least one ring associated with the resource,
or if the open ring is associated with it.

One design decision I am happy with: the `Registry` operates at the **policy layer
only**. It does not authenticate peers — it trusts the identity it receives. That is
intentional. Authentication is already handled by the transport: iroh uses QUIC with
TLS 1.3, which means every connection already carries a verified peer identity. By
the time the registry is consulted, we know *who* is asking. The registry only needs
to decide *whether* they are allowed.

This separation kept the trait clean and made it easy to implement and test.

## Pluggable backends

The crate ships two ready-made implementations.

**`InMemoryRegistry`** stores everything in memory. It is fast, has no dependencies,
and disappears when the process exits. Useful for tests, short-lived sessions, or
cases where you reconstruct state from another source on startup.

**`RedbRegistry`** persists to disk using [redb](https://crates.io/crates/redb), a
pure-Rust embedded key-value store. Ring membership and resource associations survive
restarts. This is what I use in `ringdrop` for a node that needs to remember what it
is sharing between sessions.

Swapping backends is a one-line change — the rest of the application code is generic
over `R: Registry`.

## Contract tests

How do you ensure a custom `Registry` implementation behaves correctly? The crate
ships a contract test suite alongside the trait in `crate::registry`. Any
implementor can run the full suite against their backend by calling
`registry_contract(&my_impl)` from their own test module. The tests cover the
authorization logic, edge cases around the open ring, membership changes, and
resource association.

I have seen this pattern used in a few well-known Rust projects, and I like it a lot.
For example [`tower`](https://github.com/tower-rs/tower) ships `tower-test`
alongside its `Service` trait so that custom middleware can be tested
against a consistent harness;
[`embedded-hal`](https://github.com/rust-embedded/embedded-hal) ships mock
implementations alongside its hardware abstraction traits for the same reason.
When you define a trait that has meaningful semantic invariants — not just method
signatures — shipping the tests for those invariants as part of the crate is much
better than leaving implementors to rediscover them on their own.

## Wire protocol

iroh's documentation is genuinely good and the protocol extension model is straightforward: you define your own
ALPN identifier, implement a protocol handler, and register it on the endpoint.
iroh takes care of the rest.

My approach was to define `/iroh-rings/X` as the ALPN for this protocol (where X is the incremental version number). The
`FsTransfer` implementation uses it to serve filesystem resources to authorized
peers: a remote peer requests a resource by ID, the local node checks the registry,
and if the peer is authorized the resource is streamed back. Because QUIC multiplexes
multiple streams over a single connection, `/iroh-rings/1` can coexist with other
protocols — iroh-blobs, custom application protocols — on the same connection without
any extra work.

## Code refs

- [iroh-rings on crates.io](https://crates.io/crates/iroh-rings)
- [iroh-rings on docs.rs](https://docs.rs/iroh-rings/latest/iroh_rings/)
