---
title: "Keeping GC out of the way of big iroh-blobs downloads"
slug: "iroh-blobs-gc-temp-tag"
date: 2026-06-26
draft: false
tags: ["rust", "iroh", "iroh-blobs", "open-source", "p2p", "ringdrop"]
summary: "A garbage collection race condition, when downloading large blobs: the temp_tag workaround pattern."
---

A few weeks ago, downloads of large files over [iroh-blobs](https://github.com/n0-computer/iroh-blobs) started failing intermittently. Not on every run and not on small files, but only on big blobs (and only sometimes). That combination meant to me a race condition somewhere, that I discovered being about the behavior of iroh-blobs garbage collection.

## Context

In iroh-blobs, the garbage collector (GC) is a background process that deletes blobs from the local store once nothing references them anymore, freeing disk space from files you've imported or already downloaded but no longer need.

In practice, every blob in the store is either referenced (via a tag you set explicitly or a temp_tag held by something in-progress) or it isn't. `run_gc` sweeps periodically (per `GcConfig::interval`) and unlinks the on-disk data for any hash with zero live references.

## The race condition

So, the GC wakes up on on a fixed interval to reclaim blobs that are no longer referenced. In ringdrop I had it configured at 30 seconds, which is fine for many cases, certainly not always for big files transferred over a remote computer. When a blob is *in progress*, nothing tells GC that the partial `.data` file on disk is still uncompleted.

So the GC would sweep mid-download, unlink the partial data, and the transfer would either fail or complete against a file that GC had already touched, corrupting the blob store state.

## The fix: a batch and a temp tag

`iroh-blobs` already has the needed primitives to workaround this use case: a `Batch` scope that holds open `TempTag`s, and any blob tagged inside a live batch is protected from collection for as long as the batch lives. I implemented this pattern in `Node::download_impl` in [ringdrop](https://github.com/rikettsie/ringdrop/blob/main/src/core/node.rs), the fix is to open a batch and tag the target hash before the download starts, so as to let the guard live for the duration of the transfer:

```rust
// Hold a temporary tag for the duration of the download so GC doesn't unlink
// the partial .data file while we're writing it (large files take > 30s).
let blob_batch = self
    .store
    .blobs()
    .batch()
    .await
    .context("creating download scope")?;
let _gc_guard = blob_batch
    .temp_tag(HashAndFormat { hash, format })
    .await
    .context("creating temp tag")?;

let conn = self
    .endpoint
    .connect(ticket.node_addr().clone(), iroh_rings::ALPN)
    .await
    .context("connecting to sender")?;

client
    .download(&conn, hash, format, &ticket.name, &dest, on_progress)
    .await
```

`_gc_guard` and `blob_batch` are kept alive across the whole `download_impl` call, they are indeed local variables outliving the `await`s after them. When the function returns, both drop and the hash becomes collectible again. No more racing GC on large transfers.

This pattern worked very well.

## Filing it upstream

One of the iroh maintainers ([matheus23](https://github.com/matheus23)), suggested to open a PR to highligh this behavior in the documentation, so I opened [PR #236](https://github.com/n0-computer/iroh-blobs/pull/236): a doc-only PR adding rustdoc to `Blobs::batch`, `Batch`, `Batch::temp_tag`, and `GcConfig`, explaining when and why you'd reach for this pattern.

## Refs

<p><img src="/images/icons/github.svg" class="brand-icon"> ringdrop GitHub repo: <a href="https://github.com/rikettsie/ringdrop">https://github.com/rikettsie/ringdrop</a></p>
<p><img src="/images/icons/github.svg" class="brand-icon"> iroh-blobs issue #235: <a href="https://github.com/n0-computer/iroh-blobs/issues/235">https://github.com/n0-computer/iroh-blobs/issues/235</a></p>
<p><img src="/images/icons/github.svg" class="brand-icon"> iroh-blobs PR #236: <a href="https://github.com/n0-computer/iroh-blobs/pull/236">https://github.com/n0-computer/iroh-blobs/pull/236</a></p>
</content>
