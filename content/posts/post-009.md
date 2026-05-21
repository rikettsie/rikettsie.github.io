---
title: "ringdrop: composing iroh, iroh-blobs, and iroh-rings into a P2P file drop"
date: 2026-05-21
draft: false
tags: ["rust", "iroh", "p2p", "iroh-rings", "ringdrop", "architecture"]
summary: "How to compose iroh's transport, content-addressed blob storage, and ring-based access control into a permission-aware P2P file drop."
---

The access control logic in `iroh-rings` did not start as a standalone library.
It was born inside this project, tightly coupled to ringdrop's own types and
assumptions. Only later, once the model had stabilized, did I extract it into
a separate crate through a generalization refactor — and that separation turned
out to be one of the better decisions I made along the way.

This post is about the project that came first.

That is [ringdrop](https://github.com/rikettsie/ringdrop).

The interesting part is probably how three independent
layers compose together into something useful and coherent.
Each layer does one thing and hands off to the next.

## Three layers, one node

The stack looks like this:

```
    ┌──────────────────────────────────────────────────┐
    │  iroh-rings  (who can access what)               │
    ├──────────────────────────────────────────────────┤
    │  iroh-blobs  (what is stored, how to transfer)   │
    ├──────────────────────────────────────────────────┤
    │  iroh        (how peers connect)                 │
    └──────────────────────────────────────────────────┘
```

**iroh** handles connectivity: QUIC transport, NAT hole punching, peer discovery, connection migration.
Two nodes behind separate NATs can establish a direct data path — the relay is used
for initial signaling, but once the hole is punched traffic flows peer-to-peer,
as I described in [the hole punching post](/posts/post-005).

**iroh-blobs** handles storage: content-addressed blobs identified by their BLAKE3
hash, encoded with BAO for efficient partial verification. I wrote about BAO
[earlier](/posts/post-003) — the short version is that you can verify any chunk of a
file without downloading the whole thing first.

**iroh-rings** handles access control: ring membership, resource associations, and
the authorization check before any blob is served.

`Node<R>` is the struct that owns all three. It is generic over `R: Registry`,
meaning you can swap the backend without touching anything else — the access control
policy is entirely determined by whatever `Registry` you pass in.

## RingGate: enforcement at the right layer

The `RingGate` is where access control is enforced. It sits between the incoming
blob request and the blob store: before a blob is served to a remote peer, the gate
checks the registry.

The check happens at request time, not at import time. This is intentional — ring
membership can change after a file is imported. If I add a peer to a ring, they
should immediately be able to access all resources associated with that ring, without
re-importing anything. The gate reads the current state of the registry on every
request.

A failed check is silent from the requester's perspective: the request is dropped,
no error detail is returned. Leaking *why* access was denied would tell an
unauthorized peer which resources exist — which is information you probably do not
want to give out.

## ShareTicket: out-of-band sharing

To download a file, a peer needs a `ShareTicket`. It encodes everything required:
the blob hash and the address of the node serving it.

The idea is similar to [`sendme`](https://github.com/n0-computer/sendme), the file
transfer tool by the iroh team: a self-contained ticket is all a receiver needs to
locate and request the content. The difference is what happens when the request
arrives — ringdrop's `RingGate` checks the registry before serving anything, silently
rejecting peers that do not belong to an authorized ring. The ticket gets you to the
door; the ring decides whether it opens.

The ticket is handed out of band — over a chat message, an email, whatever. The
design is deliberate: ringdrop does not have a discovery mechanism. If you have the
ticket, you know the resource exists and where to get it. If you do not have the
ticket, you cannot even ask. Combined with the ring check, this gives two independent
layers of access control: possession of the ticket, and membership in the right ring.

## Daemon and IPC

A file drop node needs to keep running between operations. I did not want to
re-initialize the iroh endpoint and reload the blob store on every CLI invocation —
that would be slow and would lose in-memory connection state.

So `ringdrop` runs as a daemon: a background process that owns the `Node` and listens
on a local TCP socket. The CLI tool (`rdrop` shell command) connects to it, sends a single
newline-terminated JSON request, and reads back a stream of JSON events until the
operation completes or fails.

The protocol is intentionally thin. Each `Op` variant maps to one operation — import
a file, create a ring, add a peer, download a blob. The daemon dispatches the request
to the node and streams progress events back. The CLI renders them.

I chose TCP over a Unix socket for platform portability (`ringdrop` runs on Linux, macOS, and Windows), and JSON over a binary protocol
because the message volume is low and debuggability matters more than throughput
in this phase. Running `nc localhost <port>` and typing a request by hand is my escape
hatch during development :)

## CLI

A typical session with `rdrop` looks like this:

```
# Start the daemon (once, keeps running)
rdrop daemon start

# Create a ring named "team" and add a peer to it
rdrop ring create team
rdrop ring add team <peer-endpoint-id>

# Import a file and associate it with the ring
rdrop import ./report.pdf --ring team

# The share ticket is printed — send it out of band
# rdrop://abcdef20...

# On the other side, a peer with the ticket downloads the file
rdrop receive <ticket>
```

The peer on the other side must be a member of the `team` ring on the serving node,
otherwise the download is silently rejected.

## What's next

There is plenty still to build. If you want to see what's coming or have a
use case to propose, the [issue tracker](https://github.com/rikettsie/ringdrop/issues)
is the right place.

Contributions are very welcome — whether that's code, documentation, or just
opening an issue with something you'd find useful.

## Refs

<p><img src="/images/icons/github.svg" class="brand-icon"> GitHub repo: <a href="https://github.com/rikettsie/ringdrop">https://github.com/rikettsie/ringdrop</a></p>
<p><img src="/images/icons/rust.svg" class="brand-icon"> Crate: <a href="https://crates.io/crates/ringdrop">https://crates.io/crates/ringdrop</a></p>
<p><img src="/images/icons/docsdotrs.svg" class="brand-icon"> Docs: <a href="https://docs.rs/iroh-rings/latest/ringdrop/">https://docs.rs/iroh-rings/latest/ringdrop/</a></p>

<p><img src="/images/icons/github.svg" class="brand-icon"> iroh-rings GitHub repo: <a href="https://github.com/rikettsie/iroh-rings">https://github.com/rikettsie/iroh-rings</a></p>
