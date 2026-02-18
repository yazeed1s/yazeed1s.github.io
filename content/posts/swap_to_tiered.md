+++
title = "From Swap to Tiered Memory: Same Idea, Different Scale"
date = 2026-02-08
description = "Comparing traditional swap with modern tiered memory systems"
[taxonomies]
tags = ["Memory", "Linux", "Performance"]
+++

Tiered memory is swap. Kind of.

You have fast memory, slow memory, and the kernel moves pages between them. That's what swap does too. But once you look closer, the differences become important.

## what swap actually does

In classic Linux memory management you have RAM (fast) and disk (very slow). When RAM is full, the kernel selects some pages and writes them to disk. That's swapping.

> Swap doesn't only kick in when RAM is completely full. How aggressively the kernel swaps depends on the `vm.swappiness` setting. At higher values, the kernel starts reclaiming anonymous pages earlier. At `swappiness=0`, it avoids swapping almost entirely until there's real memory pressure.

Later, if a swapped-out page is accessed again, you get a major page fault. The kernel reads the page back from disk into RAM.

So swap is already a two-tier system. Fast tier is DRAM, slow tier is disk. The unit of movement is still a 4KB page.

The kernel decides which pages stay in RAM and which go to disk. And that already sounds like tiered memory.

## the big difference: latency scale

The difference is scale. Rough numbers:

- DRAM: ~100ns
- CXL-attached memory: maybe ~200–300ns
- SSD: tens of microseconds
- HDD: a lifetime

Swap moves pages between nanoseconds and microseconds/milliseconds. Tiered memory moves pages between nanoseconds and slightly larger nanoseconds.

If a page sits on disk and you touch it, the program stalls hard. If a page sits in a slow memory tier, the program slows down but it might not be obvious.

## swap decisions can be coarse

Because disk is so slow, swap decisions can be rough. If a page is cold for some time, push it out. If it's accessed again, bring it back. The cost difference is so large that even simple heuristics work reasonably well.

Tiered memory doesn't have that luxury. The latency gap is smaller, so a bad migration decision won't crash performance, but small inefficiencies accumulate. Migration overhead itself becomes noticeable relative to the gap.

## hot vs cold is not binary anymore

In swap, pages are either in RAM or on disk, so cold pages go out while hot pages stay in.

In tiered memory it's more continuous. A page in the slow tier isn't dead. It's just slower.

So the question becomes: how much slower is acceptable? If a page is accessed rarely, keeping it in slow memory is fine. If it's accessed frequently but overlaps with other misses, maybe it's still fine. If it's on the critical path, it probably needs to be in fast memory.

Classification becomes more nuanced than hot vs cold. Some recent work argues that raw access count isn't enough. What matters is how much a page contributes to stall time. That depends on memory-level parallelism and overlap of misses.

## migration overhead matters more

Swapping a page to disk is expensive, but it happens relatively rarely and it's usually triggered by memory pressure.

In tiered memory, migrations can happen frequently and proactively. To migrate a page between tiers, the kernel has to allocate a new page in the target tier, copy 4KB, update page tables, possibly trigger TLB shootdowns, and synchronize across CPUs.

If migrations are too aggressive, the system spends significant time just moving pages around. Swap is reactive. Tiered memory often tries to be proactive. That increases complexity.

## granularity problems become visible

Pages are 4KB. Cache lines are 64B. With swap this mismatch didn't really matter, disk is so slow that any frequently-accessed page obviously belongs in RAM.

But tiered memory lives in a tighter performance window. A 4KB page might contain a few hot cache lines and many cold ones. Migrating the whole thing to fast memory for a small hot region wastes capacity. With huge pages (2MB) this gets worse.

## swap is mostly about capacity

Swap is fundamentally about capacity. You don't have enough RAM, so you spill to disk. If swap is heavily active, something is usually wrong.

Tiered memory is often about cost efficiency and scaling. Keep a small expensive fast tier, add a larger cheaper slow tier, try to approximate the performance of all-fast memory.

So tiered memory is more about optimization than survival. If swap is heavily active, something is usually wrong. If tiered memory is active, that's the intended design.

## similarity: same abstraction, different consequences

At the abstraction level, both swap and tiered memory move 4KB pages, update page tables, rely on page faults, and depend on kernel policies. From the kernel's perspective they're not that different.

But the consequences are. Swap mistakes cause dramatic stalls. Tiered memory mistakes cause gradual slowdowns. And gradual slowdowns are harder to detect and reason about.

## thinking forward

One thing I keep thinking about: swap worked well enough with simple heuristics because the gap was huge. Tiered memory may require more precise reasoning because the gap is smaller.

Now you care about access frequency, stall contribution, memory-level parallelism, sub-page access skew, migration stability. All of this happening at page granularity, inside a system that was designed assuming uniform memory.

In a way, tiered memory isn't a brand new idea. It's swap, compressed into the nanosecond domain. But once you compress the scale, all the small details start to matter.

## notes / random thoughts

- Swap is emergency capacity management. Tiered memory is performance optimization.
- Page abstraction survived disks. It might struggle more with heterogeneous DRAM.
- Maybe future kernels will combine tiering and scheduling more tightly.
- It's interesting that we're still moving 4KB chunks around in 2026.
- I sometimes wonder if sub-page migration will eventually become practical.
