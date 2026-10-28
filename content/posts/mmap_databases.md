+++
title = "Why Databases Stopped Using mmap"
date = 2025-12-18
description = "Letting the kernel cache your database files sounds free, and then it gets expensive."
[taxonomies]
tags = ["databases", "memory", "performance"]
+++

After writing about [buffer pools](@/posts/buffer_pools.md) the obvious question was why not skip all of it, `mmap` the data files, touch pages through pointers, and let the kernel's page cache decide what stays in memory. That deletes thousands of lines of buffer pool code, and a lot of databases tried it, LMDB, LevelDB, MonetDB, MongoDB's old engine, and SQLite has an mmap mode. Most of them backed away later.

## the kernel writes when it wants

This is the one that kills it for anything transactional. The kernel can flush a dirty mapped page whenever it wants, so a change you made through the mapping can hit disk before the transaction commits, or before its WAL record is written, and a crash at that moment leaves the database inconsistent. A buffer pool doesn't have this problem because the page sits in the database's own memory until it decides to write it, log first, data second.

People found workarounds and they're all a bit sad. MongoDB's MMAPv1 used `MAP_PRIVATE` as a copy-on-write scratch space, which means two copies of the data in memory. SQLite copies pages into user buffers before modifying them, which is most of a buffer pool again. LMDB uses shadow paging, which works but means one writer at a time. Every fix gives back the simplicity mmap was supposed to buy.

## stalls you can't schedule

When a mapped page isn't in memory, touching it faults and the thread blocks until the disk read finishes, and there's no async version of a page fault. A buffer pool can issue reads with `io_uring`, and a B+tree range scan can prefetch the next few leaf pages before it needs them. With mmap the same scan faults on every cold page, one at a time. `madvise(MADV_SEQUENTIAL)` helps a bit for scans, but it's a hint and its readahead behavior is a kernel detail that changes between versions.

## errors become signals

With a buffer pool there's one place where pages come in, so you check the checksum and handle I/O errors there. With mmap a page can be evicted and reloaded behind your back at any time, so in theory you'd verify checksums on every access, and an I/O error during that reload doesn't return anything, you get `SIGBUS`. Now your error handling is a signal handler, which is a horrible place to do recovery from.

## and it doesn't even scale

I assumed at least the performance would be good, the kernel's paging code is old and well tuned. A group at CMU (Crotty, Leis, Pavlo) benchmarked it in 2022 and it's pretty bad. In their random read test the dataset was much bigger than memory, and mmap started fine, then throughput dropped to almost nothing once the page cache filled and eviction started, and it recovered to about half of what plain `O_DIRECT` reads got. For sequential scans over 10 NVMe drives mmap stayed around 3 GB/s while direct I/O got ~60 GB/s, and mmap barely improved going from 1 drive to 10.

The reason is eviction. Each evicted page has to be removed from the TLB of every core that might have it cached, which means sending interrupts to other cores (TLB shootdowns), and under heavy eviction that happens millions of times a second. Reclaim also runs mostly in one kswapd thread per NUMA node, and every faulting thread fights over the same page table. So the bottleneck is inside the kernel's eviction path and adding faster drives doesn't move it.

## who backed out

MongoDB deprecated MMAPv1 in 2015 and removed it in 2019, InfluxDB moved off mmap after big I/O spikes once databases got past a few GB, and RocksDB started as a LevelDB fork partly because of LevelDB's mmap bottlenecks. Others (Scylla, TileDB, VictoriaMetrics) looked at it early and decided not to.

It's still fine when the whole dataset fits in memory and you mostly read, since eviction never really runs and the write ordering problem doesn't come up much, and that's roughly where LMDB lives. Past that, I think the general lesson is the same one from the buffer pool post, the kernel's page cache doesn't know about transactions or access patterns, and the database does.

## notes

- The paper is "Are You Sure You Want to Use MMAP in Your Database Management System?" (CIDR 2022), it's short and readable.
- TLB shootdowns are also what makes page migration expensive in other places, same mechanism.
- PostgreSQL never used mmap for data files, it has `shared_buffers`.
