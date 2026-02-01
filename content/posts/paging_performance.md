+++
title = "Paging to Disk and the Performance Cliff"
date = 2025-12-10
description = "Why paging exists and why it destroys performance when your working set doesn't fit."
[taxonomies]
tags = ["Operating Systems", "Memory", "Performance"]
+++

---

Virtual memory gives you the illusion of a big address space. Paging is the trick: shuffle pages between RAM and disk as needed. When it works, you barely notice. When it doesn't, your system grinds to a halt.

This post is about why that cliff exists.

---

## page faults

When a program accesses a virtual address, the MMU looks up the translation. If the page isn't in RAM right now, you get a **page fault**.

Despite the name, this isn't an error. It's the OS saying "hold on, let me go get that."

1. Program accesses memory
2. MMU checks: is this page in RAM?
3. No → triggers a page fault
4. CPU traps to kernel
5. Kernel figures out where the page actually is
6. Kernel loads it into RAM (if needed)
7. Kernel updates page tables
8. Program continues, unaware anything happened

There are two kinds:

**Minor faults.** The page is already in RAM (page cache, shared mapping, or a freshly allocated zero page). The kernel just fixes up page tables. Fast (microseconds-ish).

**Major faults.** The page has to be read from storage (swap or a file). Slow (tens of microseconds on SSDs, up to milliseconds on HDDs).

That's a 100–100,000× difference depending on what you're paging to. Minor faults are usually noise. Major faults are the problem.

Not all page faults are equal though. Some are expected:

- **Lazy allocation.** The OS doesn't actually allocate memory until you use it. First touch = minor fault.
- **Copy-on-write.** Shared pages aren't copied until someone writes to them.
- **Swapping.** Pages evicted to disk due to memory pressure. These hurt, but the system keeps working.

Others are bugs:

- Accessing invalid addresses → kernel sends SIGSEGV, process dies.
- Dereferencing null or garbage pointers → same result.

---

## paging in and out

**Paging in** = loading a page from disk into RAM when the program needs it.

**Paging out** = evicting a page from RAM to disk to make room for something else.

When RAM is full and you need to load a new page, something has to go. The OS picks a **victim page** (usually one that hasn't been accessed in a while). If it's anonymous or dirty, it has to be written somewhere first (swap or its backing file). If it's a clean file-backed page, it can often be dropped and reloaded later. Either way, you free up a frame for the new page.

```text
Before:
RAM:  [A][B][C][D] ← full
Swap: [empty]

OS needs to load page E but RAM is full
OS picks B as victim (hasn't been used recently)

After:
RAM:  [A][E][C][D]
Swap: [B]
```

If B gets accessed again later, it's a page fault. The OS loads B back in, evicts something else. This is happening constantly in the background.

Each page has a **dirty bit**. If the page has been written to since it was loaded, it's dirty. The OS can't just drop a dirty page (it has to write it to disk first). Clean pages can be discarded and reloaded from their original source.

---

## why disk hurts

Here's the thing about disk:

| Access type | Latency           |
| ----------- | ----------------- |
| RAM         | ~100 nanoseconds  |
| SSD         | ~100 microseconds |
| HDD         | ~10 milliseconds  |

SSD is 1,000× slower than RAM. HDD is 100,000× slower. When a page fault hits disk, your program stalls for an eternity in CPU terms.

One major page fault isn't a problem. A hundred per second and your application feels sluggish. A thousand, and your system is unusable.

---

## thrashing

**Thrashing** is what happens when your working set doesn't fit in RAM.

The working set is the memory your programs are actively using right now. If it exceeds available RAM, the OS has to constantly swap pages in and out. Every page you load evicts something you'll need again soon.

1. Program needs page A → page fault, load A, evict B
2. Program needs page B → page fault, load B, evict A
3. Program needs page A → page fault, load A, evict B
4. Repeat forever

Your computer spends 99% of its time moving data between RAM and disk, and 1% doing actual work. Disk light stuck on. Everything frozen. Mouse barely moves.

This is the performance cliff. You're not just slow (you're in a death spiral).

---

## example: video editing

You're editing a video project. The editor loads 20GB of project data but you only have 16GB of RAM.

The OS keeps the current timeline section in RAM (you're actively accessing it). Clips you haven't touched in 10 minutes get paged out.

When you scroll back to an old section:

- Page fault
- Disk read (you see the loading spinner)
- Some other clip gets evicted to make room

If you keep jumping around, constant paging. Everything is slow. But at least the editor works (without virtual memory, it wouldn't open at all).

This is the tradeoff. Virtual memory masks the problem, but it doesn't make it go away. You're borrowing capacity from disk, and disk is slow.

---

## this is why remote memory matters

Disk is 1,000-100,000× slower than RAM. That's the fundamental limit.

But what if you could page to someone else's unused RAM over the network instead of to disk? RDMA networks give you single-digit microsecond latencies (still slower than local RAM, but often 10–1000× faster than disk depending on whether your baseline is SSD or HDD).

That's the premise behind systems like Infiniswap: keep the paging model but replace the slow part. The cliff becomes a slope.

But that's a different post.

---

## the takeaway

Paging exists because disk is big and cheap. It lets you run programs that need more memory than you have.

But disk is also slow. When your working set exceeds RAM, every page fault is a disk access, and performance falls off a cliff.

The cliff isn't a bug. It's physics. RAM is fast, disk is slow, and no amount of clever software changes that. You can smooth the transition, but you can't eliminate it.

If you're constantly in swap, you need more RAM. Or you accept the slowdown. There's no third option (just ways to make the second option less painful).
