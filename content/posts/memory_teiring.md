+++
title = "Tiered Memory"
date = 2026-02-12
description = "Understanding tiered memory from an OS and systems perspective"
[taxonomies]
tags = ["Memory", "Hardware", "Performance"]
+++

The OS always assumed memory is uniform. Every page frame is the same speed, same cost, same latency. With CXL and tiered memory that assumption breaks. You now have fast DRAM and slower memory in the same machine.

At first it sounded simple to me. Hot pages go in fast memory, cold pages in slow memory. But from an OS perspective it gets complicated fast.

## first, what are memory pages

Before talking about tiers, pages.

Operating systems manage memory in fixed-size chunks called **pages**. On x86 that's 4KB. Sometimes you use huge pages (2MB or 1GB), but 4KB is the default.

Basically, the OS doesn't think in bytes, it thinks in pages.

When a process allocates memory, the OS maps virtual pages to physical page frames. The page table stores this mapping. CPU sees a virtual address, walks the page tables, translates it to a physical frame.

If a page is not present? Page fault. Memory full? The OS evicts pages.

So when you talk about tiered memory, what you're really asking is: which physical page frames should live in which kind of memory? That's the core question.

## why do we even need tiered memory

A server has DRAM directly attached to the CPU through memory channels. Fast, low latency, but expensive.

In large systems memory becomes a serious cost factor. In some cloud setups it's a big portion of total server cost. And many applications allocate large heaps but only actively touch part of them.

Scaling DRAM isn't trivial either. You're limited by channels, DIMM slots, signal integrity.

So the idea behind tiered memory is simple: instead of making all memory equally fast and equally expensive, have a small fast tier and a larger slower tier. Put frequently used pages in fast memory. Put less active pages in slower memory.

Conceptually simple. Implementation is not.

## CXL and heterogeneous memory

With newer interconnects you can attach extra memory that's cache-coherent but slower than local DRAM. From the OS perspective it looks like another NUMA node.

But latency is higher. Roughly:

- Local DRAM: maybe around 100ns
- Attached memory over fabric: maybe 2x or 3x that
- Still much faster than SSD or disk

So now you have heterogeneous memory inside the same system. Fast tier is local DRAM. Slow tier is attached or remote memory. Same abstraction (page), different performance.

> The "2x or 3x" latency for CXL-attached memory is a rough estimate based on early CXL 1.1/2.0 hardware. Actual latency depends on the CXL device type (Type 1, 2, or 3), the number of CXL hops, the controller implementation, and whether the access hits the device's internal cache. Some CXL memory expanders report closer to 1.5x for cached accesses. These numbers will keep changing as the hardware matures.

This breaks the old assumption that memory is uniform.

## the core problem: which pages go where

If hot pages sit in the fast tier, everything is fine. If hot pages end up in the slow tier, performance drops.

So you need to detect which pages are hot and move them.

The naive idea: count how many times each page is accessed. Most accessed pages are hot.

But it turns out that's not enough.

## hotness is not that simple

Frequency alone doesn't always tell the full story.

Modern CPUs overlap memory accesses. If several cache misses happen at the same time, the effective stall per access can be smaller. Some accesses hurt more than others, depending on timing and overlap.

So a page can be frequently accessed but not necessarily performance-critical. Another page might be accessed less often but sit directly on the critical path.

Instead of just asking "how many times was this page accessed?", you probably need to ask "how much does this page slow down the program if it's in slow memory?"

That's a harder question. I'm not sure how well current systems actually answer it. It connects OS policy with microarchitecture behavior, and I don't think the abstractions we have right now are set up for that.

## page granularity mismatch

The OS moves memory in 4KB pages. The hardware accesses memory in 64-byte cache lines.

Sometimes only a few cache lines inside a 4KB page are really hot. The rest is barely touched. If you migrate the entire page to the fast tier because a small region inside it is hot, you're wasting precious fast memory.

Huge pages make this worse. A 2MB page may contain a small hot region and a lot of cold data. Promoting the whole thing seems expensive.

There's a mismatch between OS abstraction (page-based) and real access behavior (cache-line based). Tiered memory just makes the mismatch more visible.

## migration is not free

Moving a page between tiers isn't just a pointer update. You need to allocate space in the target tier, copy 4KB of data, update page tables, possibly flush TLB entries, and coordinate across cores.

If you migrate too often, or migrate the wrong pages, you can hurt performance instead of improving it. I've seen papers where the migration overhead alone ate most of the benefit.

So it becomes a control problem. You need accurate detection, low tracking overhead, stable decisions, limited oscillation. It starts to feel like scheduling honestly. Continuously adapting to workload behavior, except the feedback signals are noisier.

## where disaggregation fits

Tiered memory is closely related to disaggregated memory.

Disaggregation means memory can be separated from compute and accessed over a fabric. That memory naturally has higher latency than local DRAM, so it often becomes the slow tier.

At that point memory management isn't just a local kernel concern. It interacts with cluster design, resource allocation, and even scheduling across machines. The boundary between OS and infrastructure gets thinner.

---

## notes / random thoughts

- Page abstraction worked well when memory was uniform. Now it feels slightly strained.
- Counting accesses is easy. Understanding performance impact is harder. I'm not convinced anyone has a great solution for this yet.
- Huge pages help TLB reach but can complicate tiering decisions.
- Migration policy starts to look like a feedback controller.
- There's always tension between transparency and giving applications more control.
- In some sense tiered memory is like swap inside RAM, but at nanosecond scale.
- It looks like a small hardware change but from the OS side it touches a lot of assumptions.
