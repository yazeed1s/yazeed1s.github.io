+++
title = "From Swap to Tiered Memory: Same Idea, Different Scale"
date = 2026-02-08
description = "Comparing traditional swap with modern tiered memory systems"
[taxonomies]
tags = ["memory", "linux", "performance"]
+++

Tiered memory is swap, kind of. You have fast memory, slow memory, and the kernel moves pages between them, which is what swap does too, but once you look closer the differences start to matter.

## what swap actually does

In classic Linux memory management you have RAM, which is fast, and disk, which is very slow, and when RAM is full the kernel selects some pages and writes them to disk, and that's swapping.

> Swap doesn't only kick in when RAM is completely full. How aggressively the kernel swaps depends on the `vm.swappiness` setting. At higher values, the kernel starts reclaiming anonymous pages earlier. At `swappiness=0`, it avoids swapping almost entirely until there's real memory pressure.

Later, if a swapped-out page is accessed again you get a major page fault and the kernel reads the page back from disk into RAM. So swap is already a two-tier system where the fast tier is DRAM and the slow tier is disk, and the unit of movement is still a 4KB page, and the kernel decides which pages stay in RAM and which go to disk, which already sounds like tiered memory.

## the big difference, latency scale

The difference is scale. Rough numbers:

- DRAM: ~100ns
- CXL-attached memory: maybe ~200–300ns
- SSD: tens of microseconds
- HDD: a lifetime

Swap moves pages between nanoseconds and microseconds or milliseconds, while tiered memory moves pages between nanoseconds and slightly larger nanoseconds. If a page sits on disk and you touch it the program stalls hard, but if a page sits in a slow memory tier the program slows down in a way that might not be obvious.

## swap decisions can be coarse

Because disk is so slow, swap decisions can be rough, so if a page is cold for some time you push it out, and if it's accessed again you bring it back, and the cost difference is so large that even simple heuristics work reasonably well. Tiered memory doesn't have that luxury, since the latency gap is smaller, so a bad migration decision won't crash performance but small inefficiencies accumulate, and the migration overhead itself becomes noticeable relative to the gap.

## hot vs cold is not binary anymore

In swap, pages are either in RAM or on disk, so cold pages go out while hot pages stay in. In tiered memory it's more continuous, since a page in the slow tier isn't dead, it's just slower. So the question becomes how much slower is acceptable, and if a page is accessed rarely then keeping it in slow memory is fine, and if it's accessed frequently but overlaps with other misses then maybe it's still fine, and if it's on the critical path then it probably needs to be in fast memory. Classification becomes more nuanced than hot versus cold, and some recent work argues that raw access count isn't enough and what matters is how much a page contributes to stall time, which depends on memory-level parallelism and overlap of misses.

## migration overhead matters more

Swapping a page to disk is expensive but it happens relatively rarely and it's usually triggered by memory pressure. In tiered memory migrations can happen frequently and proactively, and to migrate a page between tiers the kernel has to allocate a new page in the target tier, copy 4KB, update page tables, possibly trigger TLB shootdowns, and synchronize across CPUs. If migrations are too aggressive the system spends significant time just moving pages around, and where swap is reactive, tiered memory often tries to be proactive, which increases complexity.

## granularity problems become visible

Pages are 4KB and cache lines are 64B, and with swap this mismatch didn't really matter because disk is so slow that any frequently-accessed page obviously belongs in RAM. But tiered memory lives in a tighter performance window, so a 4KB page might contain a few hot cache lines and many cold ones, and migrating the whole thing to fast memory for a small hot region wastes capacity, and with huge pages of 2MB this gets worse.

## swap is mostly about capacity

Swap is fundamentally about capacity, where you don't have enough RAM so you spill to disk, and if swap is heavily active something is usually wrong. Tiered memory is often about cost efficiency and scaling, where you keep a small expensive fast tier, add a larger cheaper slow tier, and try to approximate the performance of all-fast memory. So tiered memory is more about optimization than survival, and if tiered memory is active that's the intended design rather than a warning sign.

## same abstraction, different consequences

At the abstraction level both swap and tiered memory move 4KB pages, update page tables, rely on page faults, and depend on kernel policies, so from the kernel's perspective they're not that different. But the consequences are, since swap mistakes cause dramatic stalls while tiered memory mistakes cause gradual slowdowns, and gradual slowdowns are harder to detect and reason about.

## thinking forward

One thing I keep coming back to is that swap worked well enough with simple heuristics because the gap was huge, and tiered memory may require more precise reasoning because the gap is smaller. Now you care about access frequency, stall contribution, memory-level parallelism, sub-page access skew, and migration stability, all of it happening at page granularity inside a system that was designed assuming uniform memory. In a way tiered memory isn't a brand new idea, it's swap compressed into the nanosecond domain, but once you compress the scale all the small details start to matter.

## notes / random thoughts

- Swap is emergency capacity management. Tiered memory is performance optimization.
- Page abstraction survived disks. It might struggle more with heterogeneous DRAM.
- Maybe future kernels will combine tiering and scheduling more tightly.
- It's interesting that we're still moving 4KB chunks around in 2026.
- I sometimes wonder if sub-page migration will eventually become practical.
