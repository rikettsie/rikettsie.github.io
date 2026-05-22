---
title: "BAO: verified streaming over content-addressed blobs"
slug: "bao-verified-streaming"
date: 2026-03-28
draft: false
tags: ["blake3", "content-addressable", "streaming", "iroh"]
summary: "How BAO makes BLAKE3 hashes useful for streaming: verifying data without buffering the whole file."
---

Content addressability means an object's address *is* its hash.
For example you ask for `af1349b9f5f9a1a6a...`,
you get exactly the bytes this hash points to and any deviation is immediately detectable.

The interesting engineering problem is verifying a *stream*
of bytes *as they arrive*, chunk by chunk, without buffering the whole
thing first.

BAO does this. It's an implementation of BLAKE3 hash verified streaming.

## From BLAKE3 to BAO

### The BLAKE3 tree structure

BLAKE3 hashes a file by splitting it into 1 KiB chunks, hashing each
chunk independently, then combining chunk hashes in a binary tree (a
Merkle tree), reducing pairs at each level until a single 32-byte root
hash remains.

This tree is implicit: it is not stored anywhere, it is derived from the
file length and a deterministic chunking scheme. The root hash is what
BLAKE3 reports.

### What BAO adds

**BAO** (invented by Jack O'Connor) makes the Merkle tree *explicit and
transferable*. A `.bao` file contains:

1. The file length (8 bytes)
2. All interior node hashes, in a defined pre-order layout
3. The raw file data, in 1 KiB chunks

This layout lets a parser verify any chunk as it arrives: walk the
stored tree from the root down to the chunk's position, check each
parent hash against its children, check the leaf against the chunk data.
No chunk is accepted until its ancestry back to the trusted root is valid.

This means that before a chunk's bytes are handed to the application, the parser walks up the Merkle tree from that chunk's leaf node all the way to the root, recomputing each parent hash from its two children and checking it matches the stored value. Only if every hash on that path matches, **right up to the root hash you already trust**, then the chunk is considered genuine.

If any hash on the path is wrong, the chunk is rejected, even if the chunk data itself looks fine.

## Slice extraction

The more powerful feature is **slices**: BAO can produce a proof for an
arbitrary byte range `[start, end)`. The proof is the subset of interior
nodes needed to verify that range, plus the chunks themselves. A client
can request bytes `[512 KiB, 768 KiB)` and verify them with ~2 KiB of
proof data, without downloading the rest of the file.

This is what makes BAO useful for streaming large files over untrusted
channels: video players, package managers and p2p networks can
start processing data immediately and reject corruption at the chunk level.

## Why it's fast

The proof for any range touches only O(log n) interior
nodes, where n is the total number of chunks. A 1 GiB file has roughly
1 million 1 KiB chunks, so the proof path is at most ~20 hashes deep.
Each hash is 32 bytes, the verification is few BLAKE3 compression
rounds, and the tree pre-order layout means those nodes (the hashes)
are close together on disk (no random seeks required!).

## BAO in the wild

**[iroh-blobs](https://github.com/n0-computer/iroh/tree/main/iroh-blobs)**,
the blob transfer crate in the iroh ecosystem, builds its transfer strategy
on BAO.
Blobs are identified by their BLAKE3 hash and transferred as BAO streams.
A receiver can verify each chunk in-flight, request only the ranges it needs,
and resume an interrupted transfer without re-downloading validated chunks.
The result is a verified streaming layer that works over QUIC, with no
central index required.

**[sendme](https://github.com/n0-computer/sendme)**, iroh's file-sending tool,
uses the same stack: a sender hashes
a file with BLAKE3, advertises the hash, and the receiver fetches and
verifies it as a BAO stream.

**[ringdrop](https://github.com/rikettsie/ringdrop)** is a streamed p2p
file transfer tool with ring-based access control. Peers form authorized
groups (rings) share files via `rdrop://` tickets that only ring members
can redeem. It is built on iroh for QUIC transport and NAT traversal,
and on bao-tree for verified streaming:
a BLAKE3 bitfield tracks which chunks have been verified, so an interrupted
download resumes exactly where it left off without re-transferring anything.

## Code refs

- [bao](https://github.com/oconnor663/bao) — original spec and reference implementation by Jack O'Connor; `src/encode.rs` and `src/decode.rs`
- [bao-tree](https://github.com/n0-computer/bao-tree) — production library used by iroh; `src/io/` and `src/tree.rs`
- [BLAKE3](https://github.com/BLAKE3-team/BLAKE3) — the underlying hash; `src/lib.rs`
- [iroh-blobs](https://github.com/n0-computer/iroh/tree/main/iroh-blobs) — blob transfer built on BAO; `src/store/` and `src/get.rs`

## Bibliography

- O'Connor, J. (2019). **BAO: An incremental hash format built on BLAKE3**. Specification. https://github.com/oconnor663/bao/blob/master/docs/spec.md
- O'Connor, J., Aumasson, J.-P., Neves, S., & Wilcox-O'Hearn, Z. (2020). **BLAKE3: One function, fast everywhere**. https://github.com/BLAKE3-team/BLAKE3-specs/blob/master/blake3.pdf
- Merkle, R. C. (1988). **A digital signature based on a conventional encryption function**. *Advances in Cryptology — CRYPTO '87*, LNCS vol. 293. Springer.

