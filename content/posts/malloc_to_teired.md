+++
title = "Should malloc Know About Tiered Memory?"
date = 2026-02-16
description = "glibc malloc was designed for uniform DRAM. Tiered memory changes the rules."
[taxonomies]
tags = ["OS", "memory", "linux"]
+++

This started as a side thought in advanced systems class, we were talking about OS transparency in general and I kept coming back to malloc afterward, because it's the abstraction I use the most and never actually question. Not a fully formed argument, just me chewing on it.

## malloc is blind

When you call `malloc()` you get a pointer, and the allocator has no idea if the physical page behind it is fast local DRAM or slower CXL-attached memory. glibc's allocator was built for a world where memory is mostly uniform, so it worries about arenas, chunks, `brk` vs `mmap` and lock contention, and leaves physical placement completely to the kernel. That was fine when the latency gap between memory was small, but machines now can have 2-3x differences between tiers, and from user space memory still looks flat.

## the kernel fixes it later

The way tiered memory is handled today is the kernel treats slow memory like another NUMA node, watches which pages get accessed, and moves hot pages up to fast memory and cold ones down. It works, but it's always the same shape, allocate first, observe later, migrate if needed, so the system is always reacting.

And migration isn't free, you copy pages, update page tables, flush TLB entries, and if the kernel guesses wrong about what's hot you pay all that for nothing. I keep wondering how much of this goes away if the allocator just knew _something_ before placing the memory.

## the app already knows

```c
void *hot_table = malloc(1 << 20);   // frequently accessed
void *log_buffer = malloc(1 << 20);  // rarely accessed
void *archive = malloc(100 << 20);   // mostly cold
```

Same call, same path as far as glibc is concerned, but these three are completely different in how hot they are. The application usually knows this already, and we throw that away and make the kernel figure it out again from access bits and sampling. Or even worse, trying to build AI models to handle this for us.

So one idea, not exotic at all, is to let the allocator take intent:

```c
void *malloc_hot(size_t size);
void *malloc_cold(size_t size);
```

`malloc_hot()` could bind to the fast node using what already exists like `mbind()` and `set_mempolicy()`, and `malloc_cold()` goes straight to the slow tier, so instead of allocate, detect, migrate, you just place it right the first time and skip a lot of the copying.

## is transparency still worth it here?

Virtual memory exists to hide where your bytes physically live, that's the whole point, but once the latency gap gets big enough placement starts mattering again whether we like it or not. You can keep it fully hidden and let the kernel grow more sampling and migration machinery, or leak a bit of the abstraction and let developers say hot or cold.

I don't know which is better tbh. Hints sound clean until people start labeling everything as hot because why not, and then what, the kernel ignores the hint? overrides it silently? Once you expose placement you also hand out responsibility for it, and I don't think most developers want that, even the ones who complain about performance.

Huge pages make it messier too. Inside one 2MB page only a small part might be hot, so moving the whole thing to fast memory wastes space, and the allocator doesn't even know how its allocations line up with huge page boundaries, the kernel can split or merge them later anyway. Page size, allocation and tier placement used to be separate concerns and they kinda aren't anymore.

I don't think transparency should go away, most programs don't care about this level of performance. But it doesn't have to be all or nothing, the default can stay hidden like today, and for the stuff that matters the allocator takes hints while the kernel keeps the right to override them when they're bad, or punish them!!!
