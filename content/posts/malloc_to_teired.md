+++
title = "Should malloc Know About Tiered Memory?"
date = 2026-02-16
description = "glibc malloc was designed for uniform DRAM. Tiered memory changes the rules."
[taxonomies]
tags = ["OS", "memory", "linux"]
+++

This started as a side thought during advanced systems class, we were talking about OS transparency in general and I kept coming back to malloc specifically afterward, because it's the one abstraction I interact with the most and never actually question. So this is me chewing on it after the fact, not a fully formed argument.

When you call `malloc()`, the allocator gives you a pointer. It doesn't know or care whether the physical page behind it sits in fast local DRAM or slower CXL-attached memory, and it does not even care if hugepages are enabled or not, and from user space memory still looks completely flat, one address space, no notion of near or far. Except it isn't flat anymore, not really. Machines now have 2-3x latency differences between memory tiers and the allocator is completely blind to all of it.

glibc's allocator (ptmalloc2) was built for a mostly uniform DRAM world. It manages arenas, splits and coalesces chunks, decides when to use `brk` vs `mmap`, worries about lock contention between threads, all of that. What it does not worry about is which NUMA node ends up backing an allocation, unless you explicitly ask it to. In the common path it just requests virtual memory and leaves the physical placement decision entirely to the kernel. Memory, from the allocator's point of view, is virtual address space and nothing else, it has no idea if the physical pages come from local DRAM, remote NUMA, CXL, or somewhere weirder. That was fine when latency gaps were small and mostly a bandwidth-balancing story, and it's not really fine anymore.

Tiered memory systems now treat slow memory as basically another NUMA node, demoting cold pages down and promoting hot ones back up when they get busy again. TPP migrates based on observed access frequency, and Memtis goes further and looks at access distribution, even splitting huge pages when the access pattern inside one is skewed. But notice the pattern is always the same shape, you allocate first, observe later, and migrate if needed, so we're always reacting.

And migration is not free, not even close, since you're copying 4KB pages, updating page tables, invalidating TLB entries, and maybe disturbing caches on top of it. M5's work shows misclassification and migration overhead can actively hurt performance if you're not careful with it. So you're paying a correction tax because the initial allocation was made blind. I keep wondering how much of that tax disappears if the allocator just knew *something* about what's hot before it even placed the memory.

take this:

```c
void *hot_table = malloc(1 << 20);   // frequently accessed
void *log_buffer = malloc(1 << 20);  // rarely accessed
void *archive = malloc(100 << 20);   // mostly cold
```

identical calls as far as glibc is concerned. same api, same path, no distinction whatsoever. but the temperature of these three allocations is wildly different and the allocator has zero way to express or even detect that. the application usually already knows this stuff though. a database knows its buffer pool is hot. a web server knows which structures sit in the request path. a compiler knows which tables get reused constantly. and yet we throw all of that away and make the kernel re-derive it from access bits and sampling, from scratch, every time.

so are we solving this problem too late in the pipeline? maybe.

one idea, not that exotic honestly, is to let the allocator take intent as input.

```c
void *malloc_hot(size_t size);
void *malloc_cold(size_t size);
```

`malloc_hot()` could bind to the fast NUMA node using stuff that already exists like `mbind()` and `set_mempolicy()`, nothing new needed under the hood, and `malloc_cold()` allocates straight onto the slow tier. Instead of allocate then detect then migrate you just allocate correctly the first time, which skips a bunch of migration outright, means fewer TLB shootdowns and less copying, and makes placement something you decide instead of something that gets corrected later.

which brings up the actual question underneath all of this. is OS transparency still sacred here?

virtual memory exists specifically to hide physical placement, that's the whole point of it, developers don't need to know or care where their bytes physically live, they just allocate and use. tiered memory pokes at that. once the latency gap gets big enough placement starts mattering again whether we like it or not.

you could keep full transparency, kernel infers temperature from access patterns, developers stay fully insulated, and the system just grows more internal complexity to deal with it, more sampling, more migration machinery. or you leak some of the abstraction, let developers label things hot or cold and trust them, let the allocator actually participate in placement decisions.

I genuinely don't know which is better. the second one sounds cleaner right up until you think about what happens when people misclassify things. what happens when everyone just labels everything "fast" because why not. do you override the hint? ignore it silently? once you expose placement you also hand out responsibility for it, and I don't think most application developers actually want that responsibility, even the ones who complain about performance.

huge pages make this whole thing worse, by the way, almost forgot to mention it. Memtis shows access inside a single 2MB huge page can be extremely skewed, so promoting the entire page because a tiny sliver of it is hot wastes a lot of fast-tier capacity for nothing. the allocator has no idea how its own allocations line up against huge page boundaries, and the kernel might split or merge those pages on you later anyway. page size, allocation strategy, and tier placement are all tangled together now and I don't think anyone has a clean model for how they're supposed to interact. the old layering assumed these were independent concerns. they are not, not anymore.

I don't think transparency should get abandoned entirely, it's still genuinely valuable for the vast majority of applications that don't care about squeezing out this kind of performance. but maybe it doesn't have to be all-or-nothing. default stays abstract, kernel handles it like today, and for the performance-critical stuff the allocator exposes controlled hints while the kernel keeps some override power so a bad hint doesn't destabilize the whole system.