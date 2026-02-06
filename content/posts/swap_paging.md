+++
title = "Swap and Paging: What Actually Happens When Memory Fills Up"
date = 2025-12-05
description = "How the OS shuffles memory between RAM and disk."
[taxonomies]
tags = ["Operating Systems", "Memory", "Linux", "Paging"]
+++

---

I kept hitting concepts like "page fault" and "swap" while reading memory disaggregation papers. Figured I should actually understand what these mean at a low level before going further.

---

## what swap is

Swap is disk space that acts as overflow for RAM. When physical memory fills up, the kernel moves some data to swap. Later, if that data is needed again, it gets loaded back.

That's sort of it. The complication is in the details.

---

## pages

The kernel doesn't manage memory byte by byte. Too much bookkeeping. Instead it works in fixed-size chunks called **pages**. Usually 4KB.

When you allocate memory, you get pages. When data moves to disk, it moves as pages. The physical counterpart is called a **frame**. Same size, different name. Pages are virtual, frames are physical.

8GB of RAM = roughly 2 million frames. A process might think it has way more pages than that. Most aren't backed by physical memory until used.

---

## page tables and the mmu

The CPU doesn't know about virtual addresses on its own. There's a **Memory Management Unit (MMU)** that translates virtual addresses to physical ones.

How does it know the mapping? **Page tables**. Data structures the kernel maintains. The MMU walks these to find where a virtual page actually lives (which physical frame, or if it's not in RAM at all).

Walking page tables on every memory access would be slow. So the MMU has a cache called the **TLB (Translation Lookaside Buffer)**. Recent translations are stored there. Hit the TLB = fast. Miss = pay for the walk.

Most accesses hit the TLB. That's what makes virtual memory practical.

---

## page faults

Program accesses a virtual address. MMU checks: is this page in RAM? If not, you get a **page fault**.

Not an error. Just the kernel saying "hold on, I need to go get that."

1. Program accesses memory
2. MMU finds page isn't resident
3. CPU traps to kernel
4. Kernel figures out where page lives (swap, file, or nowhere)
5. Kernel loads it into a frame
6. Page table updated
7. Program resumes

Two kinds:

**Minor fault.** Page is already somewhere in memory (page cache, shared mapping). Kernel just fixes the page table. Fast.

**Major fault.** Page has to be read from disk. Slow. Really slow.

Some faults are expected:

- **Lazy allocation.** Kernel doesn't back memory until you touch it. First access = minor fault.
- **Copy-on-write.** Shared pages aren't copied until someone writes.
- **Swapping.** Page was evicted earlier and now needed again.

Others mean bugs. Accessing garbage address = SIGSEGV.

---

## paging in and out

**Page in** = loading a page from disk into RAM.

**Page out** = moving a page from RAM to disk to free space.

RAM full. New page needed. Kernel picks a **victim** (some page not accessed recently). If it's dirty (modified since loaded), kernel writes it to swap first. If it's clean, kernel just drops it and reloads later if needed.

```text
Before:
RAM:  [A][B][C][D] ← full
Swap: [empty]

Need page E. Pick B as victim.

After:
RAM:  [A][E][C][D]
Swap: [B]
```

If B is accessed again? Major fault. Load B. Evict something else. This happens constantly under memory pressure.

---

## the dirty bit

Each page has a **dirty bit**. Set if the page has been modified since it was loaded.

Why it matters: clean pages can be dropped. Kernel can reload from file or wherever. Dirty pages can't. Kernel has to write them somewhere first.

Anonymous memory (heap, stack) that's dirty goes to swap. File-backed memory that's modified goes back to the file (or swap, depends).

---

## why disk access hurts

| Access    | Latency           |
| --------- | ----------------- |
| RAM       | ~100 nanoseconds  |
| SSD       | ~100 microseconds |
| HDD       | ~10 milliseconds  |

SSD is 1,000× slower than RAM. HDD is 100,000× slower.

Major fault = disk access = program stalls for eternity in CPU time.

One major fault, who cares. Hundred per second, app feels sluggish. Thousand, system unusable.

---

## thrashing

**Thrashing** is what happens when working set doesn't fit in RAM.

Working set = the memory you're actively using right now. Bigger than RAM? Kernel constantly swapping pages in and out. Every page you load evicts something you need again soon.

1. Need page A -> fault, load A, evict B
2. Need page B -> fault, load B, evict A
3. Need page A -> fault, load A, evict B
4. Forever

System spends 99% of time moving data. 1% doing work. This is where you get the "stuck mouse" feeling.

---

## lru: picking victims

How does kernel decide which page to evict?

Ideal: evict the one that won't be needed soonest. Can't predict future. So kernel approximates with **LRU (Least Recently Used)**. Pages not accessed in a while are good candidates.

Linux maintains active/inactive lists. Pages accessed recently go to active. Cold pages drift to inactive. Reclaim takes from inactive first.

True LRU would track exact access times for every page. Too expensive. Linux settles for "recently used" vs "not recently used."

---

## swappiness

Linux doesn't wait until RAM is completely full to start swapping. There's a knob called **swappiness**.

```bash
$ cat /proc/sys/vm/swappiness
60
```

Range is 0-200. Affects how willing kernel is to swap anonymous memory vs reclaim file cache.

- **0** = Avoid swapping. Hold app memory, sacrifice cache.
- **60** = Default. Balanced.
- **100+** = Swap more. Keep cache warm.

No universal right answer. Depends on workload.

---

## what can't be swapped

Not everything can go to disk.

**Kernel memory.** Page tables, process descriptors, driver state. Must stay in RAM. If kernel got paged out, who pages it back in?

**Pinned memory.** Memory explicitly locked by application (mlock). Used by RDMA, databases, etc.

Kernel memory leaks are dangerous. That memory is gone until reboot.

---

## overcommit

Linux lets you allocate more memory than exists. `malloc(1GB)` succeeds even with 512MB free.

Intentional. Called **overcommit**. Most programs allocate more than they use. Sparse arrays. Forked processes before exec. Refusing would break software.

Downside: actually use all that memory? OOM killer fires. Allocation succeeded, using it didn't.

```bash
$ cat /proc/sys/vm/overcommit_memory
0  # heuristic (default)

# 0 = guess what's safe
# 1 = always allow (yolo)
# 2 = strict (refuse if exceeds limit)
```

Mode 2 is safer but breaks things. Mode 1 is living dangerously.

---

## why this matters for remote memory

Main problem: disk is slow. 1,000-100,000× slower than RAM.

Systems like Infiniswap replace swap with network access to remote memory. RDMA gives single-digit microsecond latency. Still slower than local RAM. But 10-1000× faster than disk.

Keep the paging model. Replace the slow part. Cliff becomes slope.

The interesting thing: Linux has a **frontswap** interface. It's a hook that lets you intercept pages before they go to disk. Implement a few callbacks and your module becomes an alternative swap backend. That's how Infiniswap plugs into the kernel. Pages that would go to disk get redirected over the network instead.

I want to look at this interface in more detail later. How frontswap works, what the callbacks look like, and what's involved in building something like Infiniswap. Different post.

---

## notes

- Page size usually 4KB. Huge pages exist (2MB, 1GB) to reduce TLB pressure.
- "Anonymous memory" = heap, stack (no backing file). "File-backed" = mmap'd files, page cache.
- `vmstat`, `sar`, `/proc/meminfo` for monitoring paging activity.
- Swap on SSD helps. Swap on HDD is pain.
- zswap = compressed swap cache in RAM. Buys time before hitting disk.
