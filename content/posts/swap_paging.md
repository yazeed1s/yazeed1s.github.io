+++
title = "Swap and Paging: What Actually Happens When Memory Fills Up"
date = 2025-10-30
description = "How the OS shuffles memory between RAM and disk."
[taxonomies]
tags = ["OS", "memory", "paging"]
+++

I kept hitting concepts like "page fault" and "swap" while reading memory disaggregation papers, so I figured I should actually understand what these mean at a low level before going further.

## what swap is

Swap is disk space that acts as overflow for RAM: when physical memory fills up, the kernel moves some data to swap, and later, if that data is needed again, it gets loaded back. That's basically it, but the messy part is in the details.

## pages

The kernel doesn't manage memory byte by byte because the bookkeeping would be too expensive, so instead it works in fixed-size chunks called **pages**, usually 4KB.

When you allocate memory you get pages, and when data moves to disk it moves as pages. The physical counterpart is called a **frame**, same size, different name: pages are virtual, frames are physical.

8GB of RAM gives you roughly 2 million frames, and a process might think it has way more pages than that, but most aren't backed by physical memory until they're actually used.

## page tables and the mmu

The CPU doesn't know about virtual addresses on its own, there's a **Memory Management Unit (MMU)** that translates virtual addresses to physical ones using **page tables**, which are data structures the kernel maintains that the MMU walks to find where a virtual page actually lives (which physical frame, or if it's not in RAM at all).

Walking page tables on every memory access would be slow, so the MMU keeps a cache called the **TLB (Translation Lookaside Buffer)** where recent translations are stored: a hit is fast, and a miss pays for the full page table walk. Most accesses hit the TLB, and that's what makes virtual memory practical.

## page faults

When a program accesses a virtual address, the MMU checks whether that page is in RAM, and if it's not, you get a **page fault**. This isn't an error, it's just the kernel saying "hold on, I need to go get that." The program accesses memory, the MMU finds the page isn't resident, the CPU traps to the kernel, the kernel figures out where the page lives (swap, file, or nowhere), loads it into a frame, updates the page table, and then the program resumes.

There are two kinds: a **minor fault** is when the page is already somewhere in memory (page cache, shared mapping) and the kernel just fixes the page table, which is fast. A **major fault** means the page has to be read from disk, which is really slow.

Some faults are expected. **Lazy allocation** means the kernel doesn't back memory until you touch it, so the first access causes a minor fault. **Copy-on-write** means shared pages aren't copied until someone writes. **Swapping** means a page was evicted earlier and now needs to come back. Other faults mean bugs, like accessing a garbage address gives you SIGSEGV.

## paging in and out

**Page in** means loading a page from disk into RAM, and **page out** means moving a page from RAM to disk to free space.

When RAM is full and a new page is needed, the kernel picks a **victim** (some page not accessed recently). If the victim is dirty (modified since it was loaded), the kernel writes it to swap first, and if it's clean, the kernel just drops it and reloads later if needed.

```text
Before:
RAM:  [A][B][C][D] ← full
Swap: [empty]

Need page E. Pick B as victim.

After:
RAM:  [A][E][C][D]
Swap: [B]
```

If B is accessed again, you take a major fault, load B, and evict something else; this happens constantly under memory pressure.

## the dirty bit

Each page has a **dirty bit** that gets set if the page has been modified since it was loaded.

Why it matters: clean pages can be dropped because the kernel can reload them from the file or wherever they came from, but dirty pages can't be dropped without writing them somewhere first. Anonymous memory (heap, stack) that's dirty goes to swap, and file-backed memory that's been modified goes back to the file (or swap, depends).

## why disk access hurts

| Access | Latency           |
| ------ | ----------------- |
| RAM    | ~100 nanoseconds  |
| SSD    | ~100 microseconds |
| HDD    | ~10 milliseconds  |

SSD is 1,000× slower than RAM, and HDD is 100,000× slower.

A major fault means a disk access, which means the program stalls for an eternity in CPU time. One major fault, who cares. Hundred per second and the app feels sluggish. Thousand and the system is unusable.

## thrashing

**Thrashing** is what happens when the working set doesn't fit in RAM.

Working set is the memory you're actively using right now, and if it's bigger than RAM, the kernel constantly swaps pages in and out because every page you load evicts something you need again soon.

1. Need page A -> fault, load A, evict B
2. Need page B -> fault, load B, evict A
3. Need page A -> fault, load A, evict B
4. Forever

The system spends 99% of its time moving data and 1% doing work, and this is where you get the "stuck mouse" feeling.

## lru: picking victims

How does the kernel decide which page to evict? The ideal strategy would be to evict the one that won't be needed soonest, but you can't predict the future, so the kernel approximates with **LRU (Least Recently Used)** where pages not accessed in a while are good candidates.

Linux maintains active/inactive lists where pages accessed recently go to active and cold pages drift to inactive, and reclaim takes from inactive first. True LRU would track exact access times for every page, which is too expensive, so Linux settles for "recently used" vs "not recently used."

## swappiness

Linux doesn't wait until RAM is completely full to start swapping, there's a knob called **swappiness**.

```bash
$ cat /proc/sys/vm/swappiness
60
```

Range is 0-200 and it affects how willing the kernel is to swap anonymous memory vs reclaim file cache. At **0** it avoids swapping and holds app memory while sacrificing cache, **60** is the default and balanced, and **100+** means swap more aggressively and keep the file cache warm. No universal right answer, depends on workload.

## what can't be swapped

Not everything can go to disk. **Kernel memory** like page tables, process descriptors, and driver state must stay in RAM because if the kernel got paged out, who pages it back in? **Pinned memory** is memory explicitly locked by the application using mlock, and it's used by RDMA, databases, and similar systems. Kernel memory leaks are dangerous because that memory is gone until reboot.

## overcommit

Linux lets you allocate more memory than exists, so `malloc(1GB)` succeeds even with 512MB free. This is intentional, called **overcommit**, and it makes sense because most programs allocate more than they use (sparse arrays, forked processes before exec), and refusing would break software.

The downside: actually use all that memory and the OOM killer fires. Allocation succeeded, using it didn't.

```bash
$ cat /proc/sys/vm/overcommit_memory
0  # heuristic (default)

# 0 = guess what's safe
# 1 = always allow (yolo)
# 2 = strict (refuse if exceeds limit)
```

Mode 2 is safer but breaks things, and mode 1 is living dangerously.

> Mode 0 isn't blind. The kernel uses heuristics that account for total RAM, swap space, and current usage. It will still refuse obviously absurd allocations. Mode 2 enforces a strict limit based on `overcommit_ratio` (default 50%) of physical RAM plus swap. So "always allow" vs "strict" is more nuanced than it looks.

## why this matters for remote memory

The main problem is that disk is slow, 1,000-100,000× slower than RAM.

Systems like Infiniswap replace swap with network access to remote memory; RDMA gives single-digit microsecond latency, which is still slower than local RAM but 10-1000× faster than disk. Keep the paging model, replace the slow part, and the performance cliff becomes a slope.

The interesting thing is that Linux has a **frontswap** interface, which is a hook that lets you intercept pages before they go to disk. Implement a few callbacks and your module becomes an alternative swap backend, and that's how Infiniswap plugs into the kernel: pages that would go to disk get redirected over the network instead.

I want to look at this interface in more detail later, how frontswap works, what the callbacks look like, and what's involved in building something like Infiniswap. Different post.

## notes

- Page size usually 4KB. Huge pages exist (2MB, 1GB) to reduce TLB pressure.
- "Anonymous memory" = heap, stack (no backing file). "File-backed" = mmap'd files, page cache.
- `vmstat`, `sar`, `/proc/meminfo` for monitoring paging activity.
- Swap on SSD helps. Swap on HDD is pain.
- zswap = compressed swap cache in RAM. Buys time before hitting disk.
