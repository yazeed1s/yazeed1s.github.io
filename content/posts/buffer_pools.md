+++
title = "A Buffer Pool Is Just Paging in User Space"
date = 2025-12-02
description = "Database buffer pools reimplement what the OS already does. On purpose."
[taxonomies]
tags = ["databases", "memory", "OS"]
+++

A database buffer pool manages fixed-size pages in memory, decides which ones to keep and which to evict, tracks dirty pages, and writes them back to disk on its own schedule. That's the same thing the OS virtual memory system does with page frames, page tables, eviction policies, dirty bit tracking, and write-back, and the database reimplements all of it in user space on purpose.

## the OS already does this

The kernel manages physical memory in 4KB page frames and maps virtual pages to physical frames through page tables, and when memory is full it evicts cold pages to disk, so when a process touches an evicted page it faults and the kernel loads it back, and it tracks which pages are dirty and writes them back when it needs to. This is the exact same problem a database has, where it has pages on disk, some of them need to be in memory, not all of them fit, and it has to decide which pages to keep, which to evict, and when to write the dirty ones back.

So why not just let the OS handle it, map the database file with mmap and let the kernel manage everything? Some databases tried this and [it didn't go well](@/posts/mmap_databases.md).

## why databases reimplement it

The OS page cache is general purpose, so it has no concept of index pages versus temporary sort pages, no awareness that a range scan is about to need the next 50 pages, and no understanding that a dirty page has to hit the WAL before it hits the data file, and a buffer pool knows all of that.

The database builds its own page table, a hash map from `(file_id, page_number)` to a frame in the buffer pool, so when a query needs a page it looks up the hash map, and if the page is there it returns the pointer, and if not it picks a frame to evict, reads the page from disk into that frame, and updates the map. It's a page fault, just in user space and controlled entirely by the database.

## the anatomy of a buffer pool

The structure is simple. There's a frame array, a fixed-size array of page-sized slots in memory that acts as the RAM of the buffer pool. There's the page table, a hash map from page ID to frame index that translates a logical page reference into a memory location. There's an eviction policy like LRU, clock, or LRU-K that decides which frame to reclaim when the pool is full. Each frame has a dirty flag tracking whether its contents changed since it was read from disk, and a pin count tracking how many operations are currently using it, since a pinned page can't be evicted, which is the same idea as the kernel's page reference count.

When a page is requested the pool checks the page table, and if the page is already in a frame it pins it and returns the pointer, and if not it finds a victim frame through the eviction policy, writes the victim to disk first if it's dirty, reads the requested page from disk into that frame, updates the page table, and returns the pointer. It's the same flow as an OS page fault handler, just at a different layer.

## eviction, where the database knows more

The OS uses something like clock or a modified LRU that works across all processes, all files, and all pages, with no application-level knowledge, and a database can do better because it knows the access patterns. A sequential scan touches every page once, and a plain LRU would fill the cache with scan pages and evict hot index pages, which PostgreSQL avoids by using a small ring buffer for sequential scans so scan pages cycle through a handful of frames instead of polluting the whole pool. A B+tree lookup traverses root, then internal, then leaf, and the root page is hit on every lookup so it should basically never be evicted, which LRU handles naturally but a database can also just pin critical pages explicitly. Prefetching works better too, since the database knows it's doing a range scan on a B+tree and can issue async reads for the next few leaf pages before it needs them, which the OS page cache can't do because it only sees physical file offsets, not logical access patterns.

## dirty pages and write-back

This is where the difference matters most. The OS can flush a dirty page to disk whenever it wants, which is fine for normal files but dangerous for a database, because if a modified data page hits disk before the corresponding WAL record then crash recovery breaks, and the write-ahead logging rule is that the log goes first and the data page second. A buffer pool enforces this by checking, before it writes a dirty page back, that the WAL has been flushed up to the page's last modification LSN (Log Sequence Number), so the page doesn't go to disk until its log records are safe. This is impossible with mmap since the kernel has no concept of WAL ordering or LSNs and flushes when it feels like it.

## O_DIRECT and bypassing the OS page cache

Most serious databases open their files with `O_DIRECT`, which tells the kernel to skip its own page cache entirely so reads and writes go straight between the database's buffer pool and the disk. Without it you'd have the data in two places, once in the buffer pool and once in the OS page cache, which doubles the memory usage for no benefit since the database already manages caching and the OS cache is redundant. `O_DIRECT` also gives the database precise control over I/O timing with no surprises from kernel write-back threads or memory pressure from the kernel evicting buffer pool pages.

It isn't free to use though, since it requires buffers to be aligned to the filesystem block size, usually 512 bytes or 4KB, and the I/O sizes have to be aligned too, and if you get the alignment wrong the syscall fails with EINVAL, which is why most databases that use `O_DIRECT` implement their own aligned allocation routines.

PostgreSQL is an exception, since it uses the OS page cache through buffered I/O rather than `O_DIRECT` and relies on `fsync` to force data to disk, which simplifies some things but means PostgreSQL competes with the OS for memory management control.

## it's the same problem at a different layer

The parallel is almost exact:

| OS Virtual Memory                | Database Buffer Pool          |
| -------------------------------- | ----------------------------- |
| Physical page frame              | Buffer pool frame             |
| Page table (virtual to physical) | Page table (page ID to frame) |
| Page fault handler               | Buffer pool miss handler      |
| Dirty bit in PTE                 | Dirty flag per frame          |
| Reference count                  | Pin count                     |
| kswapd (page reclaim)            | Eviction policy (LRU, clock)  |
| Swap file                        | Data file on disk             |
| `write-back` flush               | WAL-ordered write-back        |

The database takes this responsibility away from the OS because general-purpose policies don't work for database workloads, where eviction needs access-pattern awareness, write-back needs WAL ordering, and prefetching needs query-plan knowledge, and the OS has none of that context.

## notes

- InnoDB (MySQL) uses a buffer pool with an LRU that splits into "young" and "old" sublists. New pages enter the old sublist, and only move to the young sublist if accessed again. This handles the scan-pollution problem.
- PostgreSQL's shared_buffers is its buffer pool. It uses a clock-sweep eviction policy.
- SQLite in WAL mode maintains its own page cache but sits on top of the OS page cache (no O_DIRECT). It works because SQLite targets small-to-medium databases where double-caching isn't expensive.
- The buffer pool is one of the first things a database student builds. It's simple in concept and brutal in the details (concurrency, latch ordering, I/O scheduling).
- Some databases are experimenting with letting the buffer pool manage allocation at finer granularity than pages. But pages have stuck around because they align with disk I/O boundaries.
