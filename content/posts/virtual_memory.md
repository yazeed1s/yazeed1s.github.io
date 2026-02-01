+++
title = "Virtual Memory Is a Lie (And That's the Point)"
date = 2025-12-05
description = "How operating systems create the illusion of infinite memory."
[taxonomies]
tags = ["Operating Systems", "Memory", "Virtual Memory"]
+++

---

Operating systems lie to programs about how much memory exists.

Every running program thinks it has gigabytes of RAM to itself. Chrome thinks it owns addresses 0x1000000 through 0x1C00000. So does your terminal. So does the video editor. But there's only one physical RAM chip, and everyone's fighting over it.

This is virtual memory. Not a bug (a feature).

---

## the illusion

When a program allocates memory, the OS doesn't hand it physical RAM addresses. It gives the program **virtual addresses** (numbers that only make sense inside that process).

The program reads and writes using these virtual addresses, mostly unaware of what's happening behind the scenes. Sometimes the data is resident in RAM. Sometimes it isn't, and the OS has to fault it in from its backing store (a file mapping or swap). The program doesn't "see" any of this directly.

```text
Chrome thinks:
"I have memory from address 0x1000000 to 0x1C00000"

Reality:
- Some of that is in actual RAM
- Some isn't resident right now (and will be faulted in from disk if needed)
- Chrome doesn't know the difference
```

This abstraction gives you three things:

**Isolation.** Process A can't access Process B's memory. Even if they use the same virtual address, they're mapped to completely different physical locations.

**Simplicity.** Every process thinks it has the whole address space to itself. No coordination needed. No worrying about stepping on another program's memory.

**Overcommit.** You can allocate more memory than physically exists. The OS figures out where to actually put it (or whether to put it anywhere at all until you use it).

---

## pages: memory in chunks

The OS doesn't manage memory byte-by-byte. Too much bookkeeping. Instead, it divides everything into fixed-size chunks called **pages**. On most systems, each page is 4KB.

So 8GB of RAM is about 2 million pages. Your program might think it has access to 4 million pages (16GB of virtual space), but only 2 million can be in physical RAM at once.

```text
Virtual Address Space              Physical RAM
+-------------------+              +-------------------+
| Page 0            | ───────────> | Frame 2           |
+-------------------+              +-------------------+
| Page 1            | ───────────> | Frame 0           |
+-------------------+              +-------------------+
| Page 2            | ──┐          | Frame 1           |
+-------------------+   │          +-------------------+
| Page 3            |   │          | Frame 3           |
+-------------------+   │          +-------------------+
                       │
                       │          Disk (Swap)
                       │          +-------------------+
                       └────────> | Swap Slot         |
                                  +-------------------+

Page 2 is "swapped out" (it exists on disk, not in RAM)
```

When referring to physical RAM, the chunks are called **frames**. Same size as pages, different name. This distinction matters when you're tracking what's virtual versus what's real.

---

## translation: the mmu

The **Memory Management Unit (MMU)** is hardware that does the virtual-to-physical address translation. When the CPU accesses memory, it asks the MMU: "what's the physical address for virtual address 0x5000?"

The OS maintains **page tables** (data structures that map virtual page numbers to physical frame numbers). The MMU walks these tables to find the answer.

But walking tables on every memory access would be slow. So the MMU uses a small cache called the **Translation Lookaside Buffer (TLB)**. Recent translations are cached here. Hit the TLB, and translation is essentially free. Miss it, and you pay for a page table walk.

```text
CPU wants address 0x5000
    ↓
MMU checks TLB
    ↓
Hit? → Return physical address (fast)
    ↓
Miss? → Walk page tables (slower)
        → Cache result in TLB
        → Return physical address
```

Most memory accesses hit the TLB. That's what makes virtual memory practical (the common case is fast).

---

## page tables: the mapping structure

The OS maintains **page tables** (data structures that map virtual page numbers to physical frame numbers). Linux uses a hierarchical structure. On x86-64 it's usually 4 levels, and can be 5 levels if 5-level paging is enabled. The names you see in Linux look like this:

```text
+-----+
| PGD | ← Page Global Directory (top level)
+-----+
   |
   |   +-----+
   +──>| P4D | ← Page Level 4 Directory
       +-----+
          |
          |   +-----+
          +──>| PUD | ← Page Upper Directory
              +-----+
                 |
                 |   +-----+
                 +──>| PMD | ← Page Middle Directory
                     +-----+
                        |
                        |   +-----+
                        +──>| PTE | ← Page Table Entry (bottom level)
                            +-----+
```

On many x86-64 systems, the P4D level is "folded" away (so you effectively have 4 levels even though the macros still exist).

Each level is an array of pointers to the next level. Why hierarchical? Two reasons:

**Saves memory.** A flat page table mapping all of virtual memory would be huge and mostly empty. With a hierarchy, you only allocate entries for memory that's actually in use.

**Enables large pages.** Higher level entries can directly map large chunks (2MB or 1GB) without going all the way to the bottom. Fewer levels = faster translation.

---

## huge pages

Linux supports **huge pages** (larger page sizes than the usual 4KB). Typically 2MB or 1GB.

Why bother? Benefits:

- **Less TLB pressure.** Fewer entries needed to cover the same memory.
- **Less page table overhead.** Higher level entries map directly, skipping levels.
- **Better performance.** For workloads with large contiguous memory access.

The tradeoff: if you only need 100KB, you still use a 2MB page. Wasted space. And finding large contiguous chunks of physical memory is harder than finding small ones.

When using huge pages, the page table walk can end early (a PMD entry can directly map a 2MB page, or a PUD entry can map 1GB). No need to go all the way down to PTE level.

---

## why overcommit works

Here's something counterintuitive: a program can allocate more memory than exists, and nothing breaks.

You have 8GB of RAM. You open Chrome (3GB), a video editor (4GB), Photoshop (3GB), plus the OS (1GB). That's 11GB for 8GB of physical RAM.

Why doesn't Photoshop refuse to open?

Because allocation isn't the same as use. When a program calls `malloc(1GB)`, the OS says "sure" and hands back virtual addresses. But it doesn't actually reserve 1GB of physical RAM (not until the program writes to those addresses).

This is **lazy allocation**. Pages exist on paper but aren't backed by real memory until touched. If the program never uses all the memory it asked for (common), the OS never has to find physical frames for it.

And if memory pressure gets high, the OS can evict cold pages and later fault them back in. The program doesn't have to know where the bytes live, but it will absolutely notice the latency when a page fault hits disk.

---

## the mental model

Think of virtual memory as a layer of indirection between programs and physical reality.

Programs see: a contiguous address space that belongs to them alone.

Reality: fragmented physical frames, shared by everyone, with some data on disk.

The OS maintains the illusion. Programs stay simple. Memory stays isolated. And you can run more stuff than your RAM should theoretically allow (at least until you actually try to use it all at once).

That's when things get interesting. But that's a different post.
