+++
title = "Infiniswap"
date = 2026-01-18
description = "How Infiniswap uses RDMA to swap pages to remote memory instead of disk."
[taxonomies]
tags = ["memory", "RDMA", "paging"]
+++

I found this paper while reading about memory disaggregation, and the idea behind it is small enough to state in one line, which is that when a machine runs out of RAM it pages to another machine's unused memory instead of to disk. What caught my attention is how they got there without asking anything from the application or the core kernel, just a kernel module that hooks into Linux's swap path so remote RAM becomes the fast tier and disk drops to being the fallback.

## the problem they're solving

Production clusters waste a lot of memory because the load is uneven, some machines are memory-starved while others sit idle, and the paper puts numbers on it, the 99th percentile machine uses 2-3× more memory than the median and over half of the cluster's aggregate memory goes unused. On the other side, when an app can't fit its working set in RAM the performance doesn't degrade gently, it falls off a cliff, and they show this with VoltDB dropping from 95K TPS to 4K and Memcached's tail latency going up 21×, mostly because disk is around 1000× slower than memory and once you're on that path you've already lost.

RDMA is what makes remote memory a plausible answer here since it gives single-digit microsecond latencies, and the remote CPU stays out of the data movement entirely because the RNIC does the DMA, so what you end up with is swap that looks normal to Linux but is backed by slabs of remote memory scattered across the cluster.

## what I thought was clever

**Using swap as the integration point.** Instead of modifying the page fault handler or remapping virtual memory, they plug into Linux's swap subsystem, which already knows how to page out and page in, so Infiniswap only has to change where those pages live. The trade-off is that you still go through the whole swap path with its page faults and context switches, but in return everything else keeps working unchanged and the thing is actually deployable.

**One-sided RDMA.** Network block devices like Mellanox's nbdX use send/recv, so the remote CPU has to wake up, copy the data, and respond, which burns multiple vCPUs on the remote machine. Infiniswap uses RDMA_READ and RDMA_WRITE instead, where the RNIC accesses remote memory directly without running any code on the remote side, so the remote CPU isn't touched at all during a transfer.

**Slab-based design.** Pages are grouped into 1GB slabs and each slab maps to a single remote machine, which keeps the metadata manageable since tracking millions of individual 4KB pages across the cluster would be expensive. Slabs doing more than 20 page I/O ops per second count as hot and get mapped to remote memory, cold slabs stay on disk.

## where it works well

The paper's evaluation shows the memory-bound workloads getting most of the benefit: Memcached stays nearly flat even when only 50% of the working set fits in memory, PowerGraph runs 6.5× faster, and VoltDB sees a 15× throughput improvement over the disk case. Across the cluster they report memory utilization going from 40% to 60%, which they frame as 47% more effective use of RAM, with network overhead staying under 1% of capacity.

## where it doesn't work

CPU-bound workloads don't get much out of it because things like VoltDB and Spark already run at high CPU utilization, so the added paging overhead of context switches, TLB flushes, and page table walks eats directly into work they were already doing, and in their tests Spark at 50% memory thrashes badly enough that it doesn't complete.

There's a limit underneath all of this that no amount of engineering removes, which is that remote memory still isn't local memory, the page faults still happen, and you're masking the latency rather than eliminating it, so for workloads where microseconds have to be predictable this is still a problem. The paper mostly assumes the masking is good enough, and for the workloads they picked it seems to be, but I'm not sure that generalizes.

## notes

- Paper: [Gu et al., "Efficient Memory Disaggregation with Infiniswap", NSDI 2017](https://www.usenix.org/system/files/conference/nsdi17/nsdi17-gu.pdf)
- Tested on 32 machines, 56 Gbps Infiniband, 64GB RAM each
- Slab placement uses "power of two choices" (pick two random machines, query free memory, use the one with more headroom)
- Slab eviction queries E+5 machines, evicts coldest (~363μs median)
- Page-out: synchronous RDMA_WRITE + async disk write (disk is fallback if remote crashes)
- Page-in: check bitmap -> RDMA_READ if remote, else disk
- Slab remapping after failure takes ~54ms (Infiniband memory registration)
- Default headroom threshold: 8GB per machine
- Hot slab threshold: 20 page I/O ops/sec (EWMA, α=0.2)
- Compared against nbdX (Mellanox), Fastswap, LegoOS
- Code: [SymbioticLab/infiniswap](https://github.com/SymbioticLab/Infiniswap)
