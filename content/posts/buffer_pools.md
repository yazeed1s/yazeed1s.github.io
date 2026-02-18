+++
title = "A Buffer Pool Is Just Paging in User Space"
date = 2025-12-02
description = "Database buffer pools reimplement what the OS already does. On purpose."
[taxonomies]
tags = ["databases", "memory", "OS"]
+++

A database buffer pool manages fixed-size pages in memory, decides which ones to keep and which to evict, tracks dirty pages, and writes them back to disk on its own schedule.

That's what the OS virtual memory system does. Page frames, page tables, eviction policies, dirty bit tracking, write-back. The database reimplements all of it. In user space.

## the OS already does this

The kernel manages physical memory in page frames (4KB). It maps virtual pages to physical frames through page tables. When memory is full, it evicts cold pages to disk. When a process touches an evicted page, it faults and the kernel loads it back. It tracks which pages are dirty and writes them back when it needs to.

This is the exact same problem a database has. The database has pages on disk. Some of them need to be in memory. Not all of them fit. The database needs to decide which pages to keep, which to evict, and when to write dirty ones back.

So why not just let the OS handle it? Map the database file with mmap and let the kernel manage everything. Some databases tried this. [It didn't go well](@/posts/mmap_databases.md).

## why databases reimplement it

The OS page cache is general purpose. It has no concept of index pages vs temporary sort pages, no awareness that a range scan is about to need the next 50 pages, and no understanding that a dirty page has to hit the WAL before it hits the data file.

A buffer pool knows all of that.

The database builds its own page table: a hash map from `(file_id, page_number)` to a frame in the buffer pool. When a query needs a page, it looks up the hash map. If the page is there, it returns the pointer. If not, it picks a frame to evict, reads the page from disk into that frame, and updates the map.

Page fault, but in user space. Controlled entirely by the database.

## the anatomy of a buffer pool

The structure is simple:

- **Frame array**: a fixed-size array of page-sized slots in memory (the "RAM" of the buffer pool).
- **Page table**: a hash map from page ID to frame index (how the database translates a logical page reference into a memory location).
- **Eviction policy**: decides which frame to reclaim when the pool is full (LRU, clock, LRU-K).
- **Dirty flag**: each frame tracks whether its contents have been modified since it was read from disk.
- **Pin count**: tracks how many operations are currently using a frame. A pinned page can't be evicted (same idea as the kernel's page reference count).

When a page is requested:

1. Check the page table. If the page is already in a frame, pin it and return the pointer.
2. If not, find a victim frame (eviction policy). If the victim is dirty, write it to disk first.
3. Read the requested page from disk into the victim frame.
4. Update the page table. Return the pointer.

That's page fault, find victim, write back if dirty, read page, update mapping. Same flow as an OS page fault handler, different layer.

## eviction: the database knows more

The OS uses something like clock or a modified LRU. It works across all processes, all files, all pages. It has no application-level knowledge.

A database can do better because it knows the access patterns.

A sequential scan will touch every page once. An LRU policy would fill the cache with scan pages and evict hot index pages. PostgreSQL handles this by using a small ring buffer for sequential scans, so scan pages cycle through a handful of frames instead of polluting the whole pool.

A B+tree lookup traverses root, internal, then leaf. The root page is accessed on every lookup. It should basically never be evicted. LRU handles this naturally, but a database can also pin critical pages explicitly.

Prefetching works better too. The database knows it's doing a range scan on a B+tree. It can issue async reads for the next few leaf pages before it needs them. The OS page cache can't do this because it only sees physical file offsets, not logical access patterns.

## dirty pages and write-back

This is where the difference matters most.

The OS can flush a dirty page to disk whenever it wants. That's fine for normal files. For a database, it's dangerous. If a modified data page hits disk before the corresponding WAL record, crash recovery breaks. This is the write-ahead logging rule: log first, data page second.

A buffer pool enforces this. Before writing a dirty page back to disk, it checks that the WAL has been flushed up to the page's last modification LSN (Log Sequence Number). The page doesn't go to disk until its log records are safe.

This is impossible with mmap. The kernel has no concept of WAL ordering or LSNs. It flushes when it feels like it.

## O_DIRECT: bypassing the OS page cache

Most serious databases open their files with `O_DIRECT`. This tells the kernel to skip its own page cache entirely. Reads and writes go straight between the database's buffer pool and the disk.

Without `O_DIRECT`, you'd have the data in two places: once in the database's buffer pool and once in the OS page cache. Double the memory usage for no benefit. The database already manages caching. The OS cache is redundant.

`O_DIRECT` also gives the database precise control over I/O timing, no surprises from kernel write-back threads or memory pressure from the kernel evicting buffer pool pages.

> `O_DIRECT` isn't free to use. It requires buffers to be aligned to the filesystem block size (usually 512 bytes or 4KB), and I/O sizes must also be aligned. If you get the alignment wrong, the syscall fails with EINVAL. This is why most databases that use `O_DIRECT` implement their own aligned allocation routines.

PostgreSQL is an exception. It uses the OS page cache (buffered I/O) rather than `O_DIRECT`, and relies on `fsync` to force data to disk. It simplifies some things but means PostgreSQL competes with the OS for memory management control.

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

The database takes this responsibility away from the OS because general-purpose policies don't work for database workloads. Eviction needs access-pattern awareness. Write-back needs WAL ordering. Prefetching needs query-plan knowledge. The OS has none of this context.

## notes

- InnoDB (MySQL) uses a buffer pool with an LRU that splits into "young" and "old" sublists. New pages enter the old sublist, and only move to the young sublist if accessed again. This handles the scan-pollution problem.
- PostgreSQL's shared_buffers is its buffer pool. It uses a clock-sweep eviction policy.
- SQLite in WAL mode maintains its own page cache but sits on top of the OS page cache (no O_DIRECT). It works because SQLite targets small-to-medium databases where double-caching isn't expensive.
- The buffer pool is one of the first things a database student builds. It's simple in concept and brutal in the details (concurrency, latch ordering, I/O scheduling).
- Some databases are experimenting with letting the buffer pool manage allocation at finer granularity than pages. But pages have stuck around because they align with disk I/O boundaries.
