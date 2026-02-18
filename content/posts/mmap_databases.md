+++
title = "Why Databases Stopped Using mmap"
date = 2025-12-18
description = "mmap looks like free I/O. It isn't."
[taxonomies]
tags = ["databases", "memory", "performance"]
+++

mmap lets you map a file into your address space and access it like memory through pointer dereferences instead of `read()` calls and user-space buffers. The OS handles paging transparently.

For a database, this looks perfect at first: map data files, access pages through pointers, let the kernel decide what stays in memory, and skip writing a buffer pool. Several databases tried it, and most backed away from it.

## why it's tempting

A traditional DBMS maintains its own buffer pool where it tracks which pages are in memory, decides what to evict, and handles I/O explicitly, which is a lot of code (thousands of lines just to manage what's cached).

With mmap you skip all that because the kernel already has a page cache, already tracks access patterns, and already evicts pages under memory pressure, so the question becomes: why duplicate it?

LMDB does it, MongoDB used to do it, LevelDB did it, MonetDB did it, and SQLite has an mmap mode, so the idea is clearly attractive.

## the transactional safety problem

The OS can flush dirty pages to disk whenever it wants. You don't control when.

If your DBMS modifies a page through the mmap'd region, that change can hit disk before the transaction commits. Crash at the wrong time and your database is inconsistent. You've violated durability, or atomicity, or both.

A buffer pool doesn't have this problem because pages live in user-space memory and the DBMS decides when to write them to disk, always writing the WAL first then the data pages, so control flow is explicit.

With mmap, you need workarounds. MongoDB's MMAPv1 engine used `MAP_PRIVATE` to create a copy-on-write workspace. Two copies of the database in memory. SQLite copies pages to user-space buffers before modifying them, which defeats the purpose of mmap. LMDB uses shadow paging, which forces single-writer concurrency.

All of these are complex. And all of them give back the simplicity that mmap was supposed to provide.

## I/O stalls you can't control

When you access an mmap'd page that's been evicted you get a page fault and the thread blocks until the OS reads the page from disk.

You can't do anything about this since there isn't an async page-fault interface to say "I'm going to need this page soon, start loading it," the thread just stops.

With a buffer pool, you control I/O explicitly. You can use `io_uring` or `libaio` for async reads. You can prefetch pages you know you'll need. A B+tree range scan can issue reads for the next few leaf pages ahead of time.

With mmap, a range scan hits a page fault on every cold page. Sequentially. Each one blocks. You can try `madvise(MADV_SEQUENTIAL)` but it's a hint, not a guarantee, and it doesn't help for non-sequential access patterns.

> `madvise` behavior varies between kernel versions and is not standardized across operating systems. On Linux, `MADV_SEQUENTIAL` triggers aggressive readahead and drops pages behind, but the readahead window size and eviction behavior are kernel implementation details that can change. Don't rely on specific behavior across versions.

## error handling gets weird

With a buffer pool error handling is centralized: you read a page, check the checksum, handle I/O errors, all in one place.

With mmap pages can be evicted and reloaded transparently, so you'd need to verify checksums on every access not just the first read. An I/O error during transparent page-in doesn't return an error code either, it raises SIGBUS, which means your error handling is now a signal handler scattered across the codebase.

If a page in your buffer gets corrupted you catch it before writing to disk, but with mmap the OS can flush a corrupted page without asking, which is silent data corruption.

## the performance collapse

This is the part that surprised me. A CIDR 2022 paper by Crotty, Leis, and Pavlo benchmarked mmap against traditional I/O and the results are bad.

Random reads on a 2TB dataset with 100GB of page cache (so ~95% of accesses are page faults):

- Traditional I/O with `O_DIRECT`: stable ~900K reads/sec
- mmap: starts fine, then collapses to near-zero when the page cache fills and eviction kicks in. Recovers to about half the throughput of traditional I/O

The collapse happens because of TLB shootdowns. When the kernel evicts a page, it has to invalidate the TLB entry on every CPU core that might have it cached. CPUs don't keep TLB entries coherent automatically. The kernel sends inter-processor interrupts—thousands of cycles each. Under heavy eviction, TLB shootdowns hit 2 million per second.

Sequential scans on 10 NVMe SSDs in RAID 0: mmap gets ~3 GB/s. Traditional I/O gets ~60 GB/s. That's 20x worse. And mmap showed basically no improvement going from 1 SSD to 10. It can't scale with modern storage bandwidth because the bottleneck is in the kernel's page eviction path, not the drives.

The page eviction itself is another problem. Linux uses a single kswapd thread per NUMA node. Under high I/O pressure it becomes the bottleneck. And the page table is a shared data structure that all threads hit during faults, creating contention.

## the graveyard

The paper tracks which databases tried mmap and what happened:

- **MongoDB** deprecated MMAPv1 in 2015, removed it in 2019. Couldn't compress data, too complex to maintain.
- **InfluxDB** replaced mmap after severe I/O spikes when databases exceeded a few GB.
- **SingleStore** found mmap calls took 10-20ms per query from write lock contention.
- **RocksDB** exists partly because LevelDB's mmap usage had performance bottlenecks.
- **TileDB, Scylla, VictoriaMetrics** all evaluated mmap during development and rejected it.

## when mmap is fine

If your entire dataset fits in memory and you're read-only, mmap works: eviction never kicks in, so you avoid TLB shootdowns, and transactional write hazards don't apply. LMDB operates in this sweet spot for some workloads.

But if your data exceeds memory, or you need writes with ACID guarantees, or you want to use fast storage at full bandwidth, mmap is the wrong tool.

## what this really comes down to

For me this comes down to one thing: the OS page cache is general-purpose while databases need very specific control. General-purpose is fine for generic workloads, but databases have specific access patterns, durability rules, and error-handling paths that the OS cannot infer.

It's similar to the tiered memory problem where the OS tries to manage page placement transparently, but transparency breaks down when the application knows something the kernel doesn't. Buffer pool vs mmap is the same tension: do you trust the OS abstraction, or do you manage things yourself because you know your workload better?

## notes

- Paper: [Crotty, Leis, Pavlo — "Are You Sure You Want to Use MMAP in Your Database Management System?", CIDR 2022](https://db.cs.cmu.edu/papers/2022/cidr2022-p13-crotty.pdf)
- Andy Pavlo has a lecture on this topic too. Worth watching if you want the full rant.
- TLB shootdowns are also a cost in page migration for tiered memory. Same mechanism, different context.
- `O_DIRECT` bypasses the page cache entirely, which is why buffer pool implementations prefer it. The DBMS manages its own cache.
- PostgreSQL has never used mmap for data access. It has its own buffer pool (shared_buffers). This was the right call.
