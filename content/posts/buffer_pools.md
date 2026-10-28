+++
title = "A Buffer Pool Is Just Paging in User Space"
date = 2025-12-02
description = "Database buffer pools reimplement what the OS already does. On purpose."
[taxonomies]
tags = ["databases", "memory", "OS"]
+++

The first time I looked at how a database buffer pool works I had this weird feeling I'd seen it before, and I had, it's the same thing the kernel does with pages, frames, page tables, eviction and dirty pages, just written again in user space. And the databases do it on purpose.

## same pieces, different names

A buffer pool is an array of page-sized frames, plus a hash map from `(file, page number)` to frame, which is the database's page table. When a query wants a page it checks the map, and if the page is there it pins it and gets a pointer back. If not, it picks a victim frame, writes it out first if it's dirty, reads the new page in, updates the map. That's a page fault handler, it just runs in the database process and never traps into anything.

Each frame has a dirty flag and a pin count, and a pinned frame can't be evicted, same idea as the kernel's reference count on a page.

| OS virtual memory   | buffer pool             |
| ------------------- | ----------------------- |
| page frame          | frame                   |
| page table          | hash map page id -> frame |
| page fault handler  | miss handler            |
| dirty bit in PTE    | dirty flag              |
| reference count     | pin count               |
| kswapd / reclaim    | eviction (LRU, clock)   |
| swap / file         | data file               |

## so why not let the OS do it

You could just `mmap` the data file and let the kernel handle all of this, and some databases tried, it didn't go well. The kernel's page cache is general purpose, so it doesn't know which pages are index pages, which are part of a scan that will never be touched again, and which can't hit disk yet.

That last one matters most I think. With write-ahead logging the log record has to be on disk before the data page it describes, so before the buffer pool writes a dirty page it checks that the WAL is flushed up to that page's LSN. The kernel has no idea what an LSN is and flushes dirty pages whenever it wants, and if it flushes a page before its log record and you crash at the wrong moment, recovery is broken. You can't fix that with a hint, you need to own the write-back.

Eviction is the other thing. A sequential scan touches every page once, and plain LRU would let that scan push out the hot index pages. PostgreSQL gives big scans a small ring of frames so they cycle through a few slots instead of polluting the whole pool, and InnoDB puts new pages in an "old" part of its LRU and only promotes them if they get touched again. And the database can prefetch the next leaf pages of a range scan because it knows it's doing a range scan, while the kernel only sees file offsets.

## O_DIRECT

If the database has its own cache and also reads through the kernel, every page lives twice in memory, once in the pool and once in the page cache. So InnoDB and others open files with `O_DIRECT` and skip the page cache completely. It's annoying to use, buffers and sizes have to be aligned to the block size or you get `EINVAL`, so everyone ends up writing their own aligned allocator.

PostgreSQL is the odd one, it still goes through the page cache and relies on `fsync`, so it double caches and shares control with the kernel. I'm not sure if that's a deliberate tradeoff or just history, probably both, and I think there's been work on direct I/O there but I don't know how far it got.

## notes

- PostgreSQL's pool is `shared_buffers`, with clock-sweep eviction.
- SQLite has its own page cache on top of the OS one and doesn't care, small databases don't suffer much from double caching.
- Concept is simple, details are brutal (latches, concurrency, I/O scheduling), and it's one of the first things people build in database courses.
