---
title: "External merge sort: sorting datasets that don't fit in RAM"
slug: "external-merge-sort"
date: 2025-05-18
draft: false
tags: ["sorting", "algorithms", "data-structures", "performance"]
summary: "How external merge sort handles datasets larger than RAM, using sorted runs and k-way merging."
---

Imagine your dataset is 200 GB, your RAM is 16 GB and you need it sorted. 

I learned the standard approach, which is unchanged in its essentials since the 1960s!

## Phase 1 — Generate sorted runs

Read as much data as fits in memory, sort it, write it back to disk.
Repeat until the whole input is covered. Each chunk written out is called
a **run**. In our example, with 16 GB of RAM you get
roughly `ceil(200 / 16) = 13` initial runs — so 13 sorted files on disk,
each 16 GB, that together cover the full dataset but are not yet merged
into a single sorted sequence. That is what Phase 2 does.

### A variant called "Replacement selection"

This is a slight improvement for the phase 1, using the trick of filling a
min-heap by reading the first 16 GB sequentially from disk.
Then, instead of writing the whole heap out and starting over
(which would be equivalent to the previous naive solution), proceed element
by element in the following way:

- pop the smallest value from the heap, write it to the current run,
and immediately read the next element from the disk input;

- if that new element is greater or equal than the last value written, it is
inserted into the heap and will eventually be written out as part of the
same run;

- if it is smaller, it is deferred to the next run. The heap stays full,
acting as a sliding window over the input rather than a fixed chunk.

On random data this variant produces runs roughly *twice* the memory size, halving
the number of initial runs. But let's continue with the phase 2.

## Phase 2 — k-way merge

Now you have `N` sorted runs on disk. Open `k` of them simultaneously,
one read buffer per run, and merge them into a single output stream using
a **min-heap of size k**. Each heap pop gives you the globally smallest
remaining element.

The choice of `k` is a hardware trade-off:

- More runs merged at once -> fewer passes over the data
- But each run needs a read buffer, and disk I/O is efficient only when
  buffers are large enough to amortize seek overhead

In practice `k` is chosen so that `k × buffer_size` fits in available
memory while leaving room for the output buffer.

## Counting the passes

With `N` initial runs and merge fan-in `k`, sorting completes in:

```
passes = ceil(log_k(N))
```

So, with 13 runs and `k = 4`, that is 2 passes. With `k = 13`, a single
merge pass is enough.

## What databases do

PostgreSQL's external sort uses replacement selection for run generation
and a k-way heap merge. SQLite does the same. MapReduce's shuffle phase
is a distributed external sort too: mappers produce sorted
partitions (runs), reducers merge them.

## References

- [PostgreSQL — tuplesort.c](https://github.com/postgres/postgres/blob/master/src/backend/utils/sort/tuplesort.c) — replacement selection and k-way heap merge
- [SQLite — vdbesort.c](https://github.com/sqlite/sqlite/blob/master/src/vdbesort.c) — external sort used by the virtual database engine
- [Apache Hadoop — MapReduce](https://github.com/apache/hadoop/tree/trunk/hadoop-mapreduce-project) — distributed external sort in the shuffle phase

- Knuth, D. E. (1973). **The Art of Computer Programming, Vol. 3: Sorting and Searching** (1st ed.), Ch. 5.4 *External Sorting*. Addison-Wesley. (2nd ed. 1998) — the canonical reference that formalized external merge sort; the algorithms themselves trace back to IBM and Bell Labs tape-sorting work circa 1959–1963.
- Vitter, J. S. (2008). **Algorithms and data structures for external memory**. *Foundations and Trends in Theoretical Computer Science*, 2(4), 305–474.
- Salzberg, B. (1989). **Merging sorted runs using large main memory**. *Acta Informatica*, 27(3), 195–215.
