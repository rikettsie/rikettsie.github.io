---
title: "Bloom filters: probabilistic membership at scale"
slug: "bloom-filters"
date: 2025-08-25
draft: false
tags: ["bloom-filter", "probabilistic", "data-structures", "rust"]
summary: "How I used a Bloom filter to check membership across 100 million URLs without blowing up RAM — and why a hash set wouldn't cut it."
---

I recently had to parse a huge structured log file containing more than
100 million lines. Each line had a timestamp, a URL and other fields.
The goal was to compute URL statistics fast, using only command line tools
(a Python script or a small Rust program), without an aggregation database
like Cassandra or ClickHouse.

The first question I needed to answer was: *have I seen this URL before?*
A hash set would be the natural answer — no counts needed, just membership.
But with 100 million distinct URLs, even a hash set is too large: each URL
string is 50–100 bytes, putting the set somewhere between 5 and 10 GB —
well beyond the available RAM in my situation.

A Bloom filter is the right tool for this: a compact probabilistic structure
that answers membership queries — "have I seen this key?" — using a fraction
of the memory a hash table would require.

## How it works

A Bloom filter is an **m-bit array**, initialised to all zeros, paired with
**k independent hash functions**, each mapping any key to a position in
`[0, m)`.

**Insertion**: hash the key k times, set all k bits.

**Query**: hash the key k times, check all k bits. If **any** is 0, the key is
definitely absent. If **all** are 1, the key is *probably* present.

There are **no false negatives**. False positives are possible: a queried
key might hash to positions all set by other keys. That rate is tunable.

But how this false positive likelihood is tuned?

## Choosing the best `m` and `k`

For `n` expected insertions and a desired false-positive rate `p`:

```
m = -n ln(p) / (ln 2)²
k = (m / n) ln 2
```

At 1% false positives, each element costs ~9.6 bits — roughly 1.2 bytes
regardless of key size. A hash set for 100 million SHA-256 URLs would
need ~3 GB; the equivalent Bloom filter needs ~120 MB.

## What's not covered

- **No deletion** in the basic structure. A deleted element might clear
  bits shared with other elements, causing false negatives. To support
  deletion you must implement **counting Bloom filters** in which presence
  bits are replaced with counters (it supports deletion at the cost of
  3×–to-4× more space).
- **No enumeration**: you cannot iterate over inserted elements. The bit
  array only records which positions were set by the hash functions, not
  the elements themselves. The mapping is one-way,
  i.e. from element to bit positions, but never the reverse.
  Moreover, multiple elements can set overlapping bits, so you cannot even
  tell which bits belong to which element.
- **False positives are permanent**: once a bit is set it stays set.

For the use case I had to tackle, none of these were a problem. 

## My solution

For this problem I used [`pybloomfiltermmap3`](https://github.com/prashnts/pybloomfiltermmap3),
a Python library that backs the bit array with a memory-mapped file — which means
the filter can be persisted across restarts and isn't constrained by available RAM alone.
The log was read as a stream through a **Python generator**, keeping file
memory flat regardless of file size.
For each URL, the filter answered my initial question in constant time;
unseen URLs were written to an output file and added to the filter.

The filter is the only data structure that grows, roughly reching
~120 MB for 100 million URLs, well within budget and orders of magnitude cheaper than a hash set.

```python
from collections.abc import Generator
from pathlib import Path

from pybloomfilter import BloomFilter


def stream_urls(log_path: Path) -> Generator[str, None, None]:
  with open(log_path) as f:
    for line in f:
      fields = line.split()
      yield fields[1]  # position of URL token in the line


def deduplicate_urls(log_path: Path, out_path: Path) -> None:
  # Initialize the Bloom filter with:
  # - how many expected elements,
  # - the desired false-positive rate (1%)
  # - path to the memory-mapped file backing the bit array
  bf = BloomFilter(100_000_000, 0.01, "urls.bloom")

  with open(out_path, "w") as out:
    for url in stream_urls(log_path):
        if url not in bf:
          bf.add(url)
          out.write(url + "\n")
```

## Code refs

- [prashnts/pybloomfiltermmap3](https://github.com/prashnts/pybloomfiltermmap3) — Python Bloom filter backed by a memory-mapped file
- [RocksDB — bloom_impl.h](https://github.com/facebook/rocksdb/blob/main/util/bloom_impl.h) — production Bloom filter used per SSTable

## Bibliography

- Bloom, B. H. (1970). **Space/time trade-offs in hash coding with allowable errors**. *Communications of the ACM*, 13(7), 422–426.
- Broder, A., & Mitzenmacher, M. (2004). **Network applications of Bloom filters: A survey**. *Internet Mathematics*, 1(4), 485–509.
- Fan, B., Andersen, D. G., Kaminsky, M., & Mitzenmacher, M. (2014). **Cuckoo filter: Practically better than Bloom**. *Proceedings of CoNEXT '14*. ACM.
- Graf, T., & Lemire, D. (2019). **Xor filters: Faster and smaller than Bloom and Cuckoo filters**. *ACM Journal of Experimental Algorithmics*, 25. (arXiv:1912.08258)
