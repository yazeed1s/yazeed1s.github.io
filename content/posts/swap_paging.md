+++
title = "Swap and Paging: What Actually Happens When Memory Fills Up"
date = 2025-10-30
description = "How the OS shuffles memory between RAM and disk."
[taxonomies]
tags = ["OS", "memory", "paging"]
+++

Swap is basically a disk pretending to be RAM, when memory fills up the kernel pushes some pages out to disk and brings them back when someone touches them again. That's the whole idea and it sounds boring, but it's the reason a machine can go from fine to frozen mouse in a few seconds, so I wanted to write down what actually happens in between.

## faults, the cheap kind and the expensive kind

When you touch a page that isn't mapped, the CPU traps and the kernel goes to find it. If the page is already somewhere in memory (page cache, a shared mapping, a page that was never touched and just needs a zero page) the kernel fixes the page table and you're back in microseconds, that's a minor fault and it happens all the time, lazy allocation and copy-on-write are built on it. If the page has to come from disk it's a major fault, and your thread just sits there.

| Access | Latency           |
| ------ | ----------------- |
| RAM    | ~100 nanoseconds  |
| SSD    | ~100 microseconds |
| HDD    | ~10 milliseconds  |

SSD is around 1000x slower than RAM and HDD around 100,000x, so one major fault doesn't matter, a hundred per second and the app feels sluggish, a thousand and the system is basically gone.

## picking a victim

When RAM is full and the kernel needs a frame, it picks some page that wasn't used recently. If it's clean (a file page nobody modified) it just drops it since it can read it again from the file, and if it's dirty it has to write it somewhere first, file pages go back to their file and anonymous memory (heap, stack) goes to swap, because it has nowhere else to live. That's kinda why swap exists at all, without it anonymous memory can never be evicted and the kernel can only throw away file cache.

The ideal would be evicting the page needed furthest in the future ([OPT](@/posts/opt.md)), but nobody knows the future, so Linux keeps active and inactive lists and reclaims from inactive first. Tracking exact access times for every page would cost too much, so it's closer to "touched recently or not" than real LRU.

## thrashing

If what you're actively using is bigger than RAM, every page you load pushes out another page you need soon, which then faults back in and pushes out the first one, forever. The system spends almost all its time moving pages and almost none doing work, and that's the stuck mouse. More swap doesn't fix it btw, it just lets you get deeper into it before the OOM killer shows up.

## knobs

`/proc/sys/vm/swappiness` (default 60, range 0-200) is how willing the kernel is to swap anonymous memory vs dropping file cache. Low keeps your app memory and sacrifices cache, high swaps more and keeps cache warm. I don't think there's a right value, people argue about this forever.

Then overcommit, which lets `malloc(1GB)` succeed with 512MB free because most programs ask for more than they touch. The allocation succeeding means nothing, the OOM killer fires later when you actually use it. Mode 0 (default) is a heuristic that still refuses obviously crazy requests, 1 always says yes, and 2 is strict accounting that breaks a lot of software.

And some memory can never be swapped at all, kernel memory (who would page the kernel back in?) and anything pinned with `mlock`, which RDMA and databases use.

## the part I actually care about

Disk is the slow part, and the paging model itself is fine. So one idea is to keep paging exactly as it is but swap to another machine's unused RAM over the network instead of disk. With RDMA that's a few microseconds, slower than local RAM but way faster than SSD, and the cliff when you run out of memory becomes more of a slope.

Linux used to have a hook for this called frontswap, where a module could catch pages on their way to swap and put them somewhere else, and remote memory systems plugged in there. I think it got removed a couple of years ago (6.5?) and zswap now hooks in directly, so I'm not sure what the clean way to do this is today. Need to look into it, different post.

## notes

- Pages are usually 4KB, huge pages (2MB, 1GB) exist to cut TLB misses.
- `vmstat`, `sar`, `/proc/meminfo` to watch paging. `si`/`so` in vmstat is swap in/out.
- zswap compresses pages in RAM before they hit swap, buys some time.
- Swap on SSD is fine, swap on HDD is pain.
