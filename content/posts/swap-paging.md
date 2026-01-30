+++
title = "OS Swap and Paging: How Virtual Memory Works"
date = 2025-01-30
description = "How operating systems use paging and swap space to give applications more memory than physically exists."
[taxonomies]
tags = ["Operating Systems", "Memory", "Virtual Memory", "Linux"]
+++

---

Operating systems provide applications with the illusion of having more memory than physically available. They do this through virtual memory - a layer of abstraction between what applications see and what actually exists in RAM. When physical memory runs out, the OS pages data out to disk. Here's how it works.

---

## Virtual Memory Basics

Every process gets its own virtual address space. When your program allocates memory or accesses a variable, it uses virtual addresses - not physical RAM addresses. The OS and CPU translate these virtual addresses to physical ones transparently.

```
    Application View              Physical Reality
    +------------------+          +------------------+
    |                  |          |                  |
    | Virtual Address  |          |   Physical RAM   |
    |  Space (large)   |   --->   |    (limited)     |
    |                  |          |                  |
    +------------------+          +------------------+
                                          +
                                          |
                                          v
                                  +------------------+
                                  |    Swap Space    |
                                  |     (disk)       |
                                  +------------------+
```

This abstraction gives you:

1. **Isolation** - Process A can't access Process B's memory
2. **Simplicity** - Every process thinks it has the whole address space to itself
3. **Overcommit** - You can allocate more memory than physically exists

## Pages and Frames

Memory is divided into fixed-size chunks:

- **Virtual pages** - chunks of the virtual address space (typically 4KB)
- **Physical frames** - chunks of actual RAM (same size as pages)

The page table maps virtual pages to physical frames. Not all virtual pages need a physical frame at any given time - that's the key insight.

```
    Virtual Address Space              Physical RAM
    +-------------------+              +-------------------+
    | Page 0            | -----------> | Frame 2           |
    +-------------------+              +-------------------+
    | Page 1            | -----------> | Frame 0           |
    +-------------------+              +-------------------+
    | Page 2            | --+          | Frame 1           |
    +-------------------+   |          +-------------------+
    | Page 3            |   |          | Frame 3           |
    +-------------------+   |          +-------------------+
    | ...               |   |
    +-------------------+   |          Swap Space (Disk)
    | Page N            |   |          +-------------------+
    +-------------------+   +--------> | Swap Slot 0       |
                                       +-------------------+
                                       | Swap Slot 1       |
                                       +-------------------+

    Page 2 is "swapped out" - it exists on disk, not in RAM
```

## Page Faults

A page fault occurs when a process tries to access a virtual address whose page isn't currently in physical memory. This isn't an error - it's a normal part of how virtual memory works.

When a page fault happens:

1. CPU traps to the kernel
2. The Virtual Memory Manager (VMM) looks up where the page lives
3. If it's on disk (in swap), the VMM reads it back into a free frame
4. The page table is updated to point to the new frame
5. The process resumes, unaware anything happened

```
    Process accesses virtual address 0x7fff1234
                    |
                    v
    +----------------------------------+
    | CPU checks page table            |
    | Page not present? PAGE FAULT     |
    +----------------------------------+
                    |
                    v
    +----------------------------------+
    | Kernel takes over                |
    | Where is this page?              |
    +----------------------------------+
           /                \
          /                  \
    In swap space         Never allocated
         |                    |
         v                    v
    Read from disk      Zero-fill or
    into free frame     allocate new frame
         |                    |
         +---------+----------+
                   |
                   v
    +----------------------------------+
    | Update page table                |
    | Resume process                   |
    +----------------------------------+
```

There are two types of page faults:

- **Minor fault** - The page is already in memory (shared library, buffer cache). Just update the page table.
- **Major fault** - The page must be read from disk. This is slow.

## Paging Out (Swapping)

When physical memory fills up and the OS needs to make room for new pages, it has to evict existing ones. This is called paging out or swapping.

The OS picks a victim page (usually one that hasn't been accessed recently), writes it to swap space, and marks it as "not present" in the page table. The physical frame is now free.

```
    Before:                             After:
    +-------------+                     +-------------+
    | Frame 0: A  |                     | Frame 0: A  |
    +-------------+                     +-------------+
    | Frame 1: B  | <-- victim          | Frame 1: D  | <-- new page
    +-------------+                     +-------------+
    | Frame 2: C  |                     | Frame 2: C  |
    +-------------+                     +-------------+

    Swap Space:                         Swap Space:
    +-------------+                     +-------------+
    | (empty)     |                     | B           | <-- evicted
    +-------------+                     +-------------+
```

## Page Replacement Algorithms

How does the OS choose which page to evict? Common strategies:

1. **LRU (Least Recently Used)** - Evict the page that hasn't been accessed for the longest time. Good in theory, expensive to track perfectly.

2. **Clock (Second Chance)** - Approximate LRU. Each page has an "accessed" bit. The algorithm sweeps through pages like a clock hand, clearing the bit. Pages with the bit already cleared get evicted.

3. **LRU-K** - Consider the last K accesses, not just the most recent one.

Linux uses a variant with two lists: active and inactive. Pages start on the inactive list and get promoted to active if accessed again. Eviction happens from the inactive list.

## Swap Space

Swap space is where paged-out data lives on disk. It can be:

- A dedicated partition (`/dev/sda2`)
- A regular file (`/swapfile`)
- Multiple swap areas with priorities

```bash
# Check swap status
$ swapon --show
NAME      TYPE SIZE USED PRIO
/swapfile file  8G 1.2G   -2

# View memory and swap usage
$ free -h
              total   used   free  shared  buff/cache  available
Mem:           16G    8G     2G    500M        6G         7G
Swap:           8G   1.2G   6.8G
```

Any block device that implements the expected interface can serve as swap. The kernel doesn't care if it's an SSD, HDD, or something more exotic (like remote memory - but that's another post).

## Thrashing

Thrashing happens when the system spends more time paging than doing actual work. The working set of active processes exceeds physical RAM, so pages constantly get evicted and immediately faulted back in.

Signs of thrashing:

- High swap I/O (`vmstat`, `iostat`)
- System becomes unresponsive
- Load average spikes

Fixes:

- Add more RAM
- Kill memory-hungry processes
- Reduce the number of running processes
- Use faster storage for swap (SSD > HDD)

## Swappiness

Linux has a tunable called `swappiness` (0-100) that controls how aggressively the kernel swaps:

- **swappiness = 0** - Only swap to avoid OOM
- **swappiness = 100** - Swap aggressively
- **Default = 60** - Balanced

```bash
# Check current value
$ cat /proc/sys/vm/swappiness
60

# Change temporarily
$ sudo sysctl vm.swappiness=10

# Change permanently (add to /etc/sysctl.conf)
vm.swappiness=10
```

Lower values keep more application data in RAM at the expense of file system cache. Higher values reclaim application memory earlier to keep more file cache.

## The OOM Killer

When both RAM and swap are exhausted and the system can't free any more memory, Linux invokes the OOM (Out of Memory) killer. It picks a process to kill based on:

- How much memory it's using
- How long it's been running
- Its `oom_score_adj` value (can be tuned per-process)

The goal is to free enough memory to keep the system alive. It's a last resort.

```bash
# Check a process's OOM score (higher = more likely to be killed)
$ cat /proc/<pid>/oom_score

# Protect a process from OOM killer (-1000 to 1000)
$ echo -500 > /proc/<pid>/oom_score_adj
```

## Why This Matters

Understanding swap and paging helps you:

1. **Size your systems correctly** - Know when you're overcommitting
2. **Debug performance issues** - Recognize when swapping is the bottleneck
3. **Tune appropriately** - swappiness, OOM scores, swap size

Modern systems have a lot of RAM, but swap still matters. It's the safety net that keeps your system from crashing when memory pressure spikes.

---

## Summary

- Virtual memory gives processes the illusion of large, isolated address spaces
- Pages (4KB chunks) map virtual addresses to physical frames
- Page faults bring pages into RAM on demand
- When RAM fills up, the OS pages out to swap space
- Thrashing occurs when working set exceeds available RAM
- Linux provides tuning knobs: swappiness, OOM scores, swap priority

Understanding this foundation is essential before diving into more advanced topics like memory disaggregation and remote paging.
