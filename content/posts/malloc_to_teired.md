+++
title = "Should malloc Know About Tiered Memory?"
date = 2026-02-16
description = "glibc malloc was designed for uniform DRAM. Tiered memory changes the rules."
[taxonomies]
tags = ["OS", "Memory", "Linux", "glibc"]
+++

When you call `malloc()`, the allocator gives you a pointer. It doesn't know or care whether the physical page behind it sits in fast local DRAM or slower CXL-attached memory. From user space, memory still looks flat. But it isn't anymore. Machines now have 2–3x latency differences between memory tiers, and the allocator is completely blind to that.

---

## how glibc malloc sees the world

glibc's allocator (ptmalloc2) was designed in a mostly uniform DRAM world. It manages arenas, splits and coalesces chunks, decides when to use `brk` and when to use `mmap`, and tries to reduce lock contention between threads. But it doesn't care about which NUMA node backs an allocation unless the application explicitly asks for it. In the common case, it just requests virtual memory and leaves physical placement to the kernel.

So from the allocator's perspective, memory is virtual address space. It doesn't know whether the physical pages will come from local DRAM, remote NUMA, CXL-attached memory, or something else. That blindness was perfectly reasonable when latency differences were small and mostly about bandwidth balancing. The allocator could afford to ignore placement because the hardware was close to uniform.

---

## what tiered memory changes

In tiered memory systems, the kernel often treats slow memory as another NUMA node. It may demote cold pages to slow memory and promote hot pages back to fast DRAM. Research systems like TPP migrate pages based on observed access frequency, and Memtis tries to improve classification by looking at access distribution and even splitting huge pages when access inside them is skewed.

But the pattern is always the same: allocate first, observe later, migrate if needed. The allocator places data somewhere, the kernel watches page faults or samples accesses, then corrects the placement. We're always reacting.

Migration isn't free. It involves copying 4KB pages, updating page tables, invalidating TLB entries, and potentially disturbing caches. Work like M5 shows that misclassification and migration overhead can actually hurt performance if not handled carefully. So you're paying a correction cost because the initial allocation was blind. I keep wondering how much of this cost could be avoided if the allocator had any information at all about what's hot.

---

## the allocator knows nothing about temperature

Consider a simple program:

```c
void *hot_table = malloc(1 << 20);   // frequently accessed
void *log_buffer = malloc(1 << 20);  // rarely accessed
void *archive = malloc(100 << 20);   // mostly cold
```

From glibc's perspective, these are identical calls. Same API, same path. But their temperature is completely different. The allocator has no way to express or detect that difference.

The application often already knows which data is critical. A database knows its buffer pool is hot. A web server knows which structures sit in the request path. A compiler knows which tables are heavily reused. Yet we force the kernel to guess using access bits and heuristics.

That naturally leads to the question: are we solving the problem too late?

---

## should malloc become tier-aware?

One idea, not that exotic: let the allocator express intent.

```c
void *malloc_hot(size_t size);
void *malloc_cold(size_t size);
```

Internally, `malloc_hot()` could bind memory to the fast NUMA node using mechanisms that already exist, like `mbind()` or `set_mempolicy()`. `malloc_cold()` could allocate directly on the slow tier. Instead of allocate -> detect -> migrate, you'd allocate correctly from the start.

This avoids some migration entirely. Fewer TLB shootdowns, less page copying. Placement becomes a proactive decision rather than a reactive correction.

But now the deeper question comes up.

---

## is OS transparency still sacred?

Virtual memory was designed to hide physical placement. That abstraction is powerful because developers don't need to care where bytes live. They just allocate and use.

Tiered memory challenges that. When latency differences become large enough, placement starts to matter again.

You can keep full transparency. The kernel observes access patterns and tries to infer temperature. Developers stay insulated. The system grows more complex internally, with more sampling and migration.

Or you can leak some abstraction. Let developers label allocations as hot or cold. Trust applications to express intent, and let the allocator participate in placement.

I honestly don't know which is better. The second approach sounds cleaner until you think about what happens when developers misclassify. What if everything gets labeled "fast"? Do you override their hints? Ignore them? Once you expose placement, you also expose responsibility, and most application developers probably don't want that.

---

## huge pages make it worse

Memtis shows that access inside a 2MB huge page can be highly skewed. Promoting an entire huge page to fast memory because a small region is hot wastes precious capacity. The allocator doesn't know how its allocations align with huge pages. The kernel may split or merge them later.

So page size, allocation strategy, and tier placement are all interacting and I'm not sure anyone has a clean model for how they should interact. The original layering between allocator and kernel assumed these things were independent. They're not anymore.

---

## a possible middle ground

I don't think we should abandon OS transparency completely. It's still valuable, especially for most applications that don't care about deep performance tuning. But maybe transparency should become adjustable.

By default, memory stays abstract. The kernel handles tiering. But for performance-critical systems, the allocator could expose controlled hints, and the kernel could enforce limits so that misclassification doesn't destabilize things.

I don't know if this would hold up in real systems. It depends on how well the kernel can override bad hints, and on whether developers will bother annotating allocations. My guess is most people ignore it and a small group gets real value out of it.
