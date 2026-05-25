---
title: "Extending the resource access permission model on iroh-rings"
slug: "iroh-rings-access-permission-data-model"
date: 2026-05-25
draft: false
tags: ["rust", "iroh-rings", "p2p", "access-model", "permissions", "iroh"]
summary: "iroh-rings started with a binary ALLOWED/DENIED access model. Reasoning here the transition to a READ/WRITE/DELETE permission model."
---

[iroh-rings](https://github.com/rikettsie/iroh-rings) is a Rust library I am writing to add a very intuitive ring-based access control for sharing resources over [iroh](https://iroh.computer), an awesome QUIC/TLS peer-to-peer networking stack i love.
A **ring** is a named group of peers. Resources are associated with rings, and every member of a ring gets access to those resources. The library was originally built as the access layer for the `ringdrop` file-sharing application, so the first version of the access model was simple: a peer either had access to a resource or not.

That binary model worked well for downloading. But as the scope of iroh-rings grew beyond file downloads, the question became obvious: *what kind of access to resources?*

## The old model

In `v0.4`, the registry answered a single question:

```rust
fn is_allowed(&self, peer: &EndpointId, resource_id: &ResId) -> Result<bool, Error>;
```

A peer was allowed or it wasn't. This was enough for reading — but a peer with write intent or delete intent would get the exact same `true` answer as a read-only peer. Any finer distinction had to be pushed into application code, outside the library. That boundary was leaking — a sign that the two components weren't respecting single responsibility.

## Thinking about the right operations

The first instinct was to lift the CRUD model from REST APIs - I've written so many classic web services with that pattern that it came naturally: create, read, update, delete. But a P2P context is not a REST API.

In iroh-rings, "create" and "update" means granting Write permission for a (potentially new) resource ID on a ring. The payload transfer is handled at the application layer via the `Transfer` implementation — the protocol itself only tracks IDs and permissions.

Here is the permission model:

| Permission | What it authorises |
|---|---|
| `Read`   | Get the resource (download bytes) |
| `Write`  | New push or overwrite the resource, but only into rings the peer already belongs to |
| `Delete` | Remove the ring–resource association (the underlying data is untouched) |

### Notes on modifying permissions (`Write` and `Delete`)

A peer in a ring with `Write` permission on a resource can push updates to that resource (peer's ring membership is required).

`Delete` is narrower than it sounds. Removing a ring–resource association does not destroy the data. True deletion is a local-only operation reserved for the registry owner. From a remote peer's perspective, `Delete` means "you may ask me to stop serving this resource from this ring".

## Where permissions live

Permissions are attached to each **ring–resource association** independently. Ring `A` can grant `Read` on resource `X` but `Read + Write` on resource `Y` — there is no global "ring `A` grants `Read`"; the permission is always scoped to a specific (ring, resource) pair.

A peer's effective permissions on a resource are the union across all rings they belong to that are associated with that resource.

I explicitly chosed not to support per-peer permission overrides. The argument for them seemed appropriate at the first look: "what if I want one peer in a ring to have less access than the others?" But per-peer overrides introduce a second access-control dimension that interacts with ring membership in subtle ways. This pushes the system toward a full RBAC or ABAC model, with all the corresponding implementation complexity and audit surface. For this library, the cognitive overhead seemed not worth it. If different peers need different permissions on the same resources, they can be simply put in different rings containing them.

## The open ring constraint

iroh-rings has a built-in ring called `"open"`. Any resource associated with the open ring is readable by *any* peer, regardless of ring membership. Think of it as the public-read ACL of the library.

The open ring only makes sense for `Read`. Allowing `Write` or `Delete` via the open ring would mean any peer on the network could modify or deassociate resources — that is not a permission, it is an absence of access control. The library enforces this at the validation layer:

```rust
pub(crate) fn validate_ring_permissions(
    ring_name: &str,
    permissions: &[Permission],
) -> Result<(), Error> {
    if permissions.is_empty() {
        return Err(Error::EmptyPermissionSet);
    }
    if ring_name == OPEN_RING_NAME
        && permissions.iter().any(|p| !matches!(p, Permission::Read))
    {
        return Err(Error::OpenRingReadOnly);
    }
    Ok(())
}
```

The open ring and private rings may coexist on the same resource — a resource can be publicly readable via the open ring while remaining writable or deletable only by members of a private ring.

## The wire protocol

Giving permissions meaning requires transmitting intent. The iroh-rings wire protocol is minimal: the initiating peer sends a length-prefixed resource id, and the gate replies with a single status byte. To carry the permission, we added one byte after the resource id — the **operation byte** — and bumped the ALPN from `/iroh-rings/1` to `/iroh-rings/2`:

```
[2B resource_id_len_LE][N bytes resource_id][1B operation] → [1B status]
```

The gate reads the operation byte, maps it to a `Permission`, checks the registry, and either grants or denies the stream:

```rust
pub fn encode_request(resource_id: &[u8], permission: Permission) -> Result<Vec<u8>, Error>;
```

## How permissions are stored

Each ring–resource association carries its own permission set, encoded as a compact bitfield: bit 0 = Read, bit 1 = Write, bit 2 = Delete. A single byte can represent any combination. Checking a permission is a bitwise AND — no allocation, no iteration over a `Vec<Permission>`:

```rust
let pbit = permission_bit(permission); // 0b001 / 0b010 / 0b100
if bits & pbit == 0 {
    continue; // this ring does not grant the requested permission
}
```

Both the in-memory and redb backends share the same ring-computation logic via `compute_resource_rings`, so any change to the model propagates to both automatically. In the redb backend specifically, the bitfield is stored in a dedicated `RESOURCE_RING_PERMS` table keyed by a composite of resource id and ring name:

```
[2B resource_id_len_LE][resource_id bytes][ring_name bytes] → u8 bitfield
```

The 2-byte length prefix makes the key unambiguous regardless of the content of the resource id.

## What the model does not do

iroh-rings enforces **what** is allowed. It does not authenticate **who** is speaking — that is the job of the transport. QUIC/TLS 1.3 completes a handshake before any application byte is exchanged; each peer's `EndpointId` is their Ed25519 public key, and completing the handshake proves possession of the corresponding private key. By the time the gate consults the registry, the caller's identity is cryptographically guaranteed.

The registry also does not model content-level access (e.g., byte ranges, file paths, version numbers). That belongs in the `Transfer` sub-protocol, which runs after the gate has granted access. iroh-rings draws a clean line between *who may do what on which resource* and *what is actually exchanged*.

iroh-rings is deliberately a simple group-based access control library. There is no delegation, no "I temporarily grant you access." A peer either belongs to a ring or it doesn't. If you need signed capability delegation, [rcan](https://github.com/n0-computer/rcan) is the composable layer for that: it implements attenuated capability tokens with delegation chains and expiration. The two can coexist cleanly — a ring member could issue a time-bound `Write` token via rcan to an external peer, without ever adding that peer to the ring.

iroh-rings remains the authority on group membership and static permissions; rcan handles the ephemeral, delegated layer on top.

## Conclusion

The move from binary `is_allowed` to permission-typed `has_permission` is a small API surface change with meaningful consequences for what iroh-rings can express. The three-permission model covers the operations that genuinely matter, at the registry layer, in many use-cases.

Check the sources for this new [iroh-rings v0.5](https://github.com/rikettsie/iroh-rings), available on [crates.io](https://crates.io/crates/iroh-rings). Feedback welcome.
