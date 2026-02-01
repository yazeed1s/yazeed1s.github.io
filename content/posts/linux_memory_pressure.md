+++
title = "What Linux Does When Memory Runs Out"
date = 2025-12-15
description = "How Linux handles memory pressure (and why it sometimes kills your processes)."
[taxonomies]
tags = ["Linux", "Operating Systems", "Memory"]
+++

---

When you're out of RAM and swap is filling up, Linux has to make hard choices. This post is about what those choices are and why they sometimes end with your process getting killed.

---

## page replacement

When RAM is full and something new needs to come in, something else has to go out. Linux uses LRU lists (active/inactive), split for anonymous memory (heap/stack) and file-backed pages (page cache). The mental model is the same:

**Active list.** Pages that have been accessed recently. These are "hot" (probably still in use).

**Inactive list.** Pages that haven't been touched in a while. These are candidates for eviction.

New pages usually start inactive. If they get accessed again, they get promoted to active. Reclaim prefers victims from inactive.

This is a rough approximation of LRU (Least Recently Used). True LRU would track exact access times for every page, which is too expensive. Linux settles for "recently used" versus "not recently used."

When memory pressure rises, the kernel scans pages on the inactive list, looking for victims. Clean pages (unchanged since loaded) can be dropped immediately. Dirty pages have to be written to disk first.

---

## swap behavior

Swap is the overflow area (disk space where evicted pages go).

Linux doesn't wait until RAM is completely full to start swapping. It swaps based on a tunable called **swappiness** (0-200). Roughly, it's a knob for how willing the kernel is to swap anonymous memory versus reclaim file cache.

- **0** = Avoid swapping anonymous memory unless you have to
- **60** = Default-ish, balanced
- **100+** = Swap more aggressively to keep RAM free for file cache

```bash
$ cat /proc/sys/vm/swappiness
60
```

Higher swappiness means Linux will push application pages to swap earlier to keep more room for file system cache. Lower swappiness keeps your applications in RAM longer but may hurt cache performance.

There's no universally right answer. It depends on whether your workload benefits more from cached file data or from keeping application memory resident.

---

## what can't be swapped

Not all memory can go to disk.

**Kernel memory can't be swapped.** The kernel's own data structures (page tables, process descriptors, driver state) must stay in RAM. If the kernel itself got paged out, who would page it back in?

Specifically, a lot of kernel allocations live in the direct map (the linear mapping of physical memory into kernel virtual addresses). It's still mapped via page tables, but it's not pageable like user memory.

This is why memory leaks in kernel code are especially dangerous. That memory is gone until reboot.

---

## the oom killer

When both RAM and swap are full and the kernel can't reclaim any more memory, Linux invokes the **OOM Killer** (Out Of Memory Killer).

It picks a process and terminates it. No negotiation. Just dead.

How it chooses:

- **Memory usage.** Bigger hogs are more likely targets.
- **OOM score.** Each process has a computed score, plus a tunable adjustment, that influences selection.

```bash
# Check a process's OOM score
$ cat /proc/1234/oom_score
582

# Protect important processes
$ echo -1000 > /proc/1234/oom_score_adj
```

Setting `oom_score_adj` to -1000 makes a process essentially unkillable by the global OOM killer. Set it to +1000 and it becomes a preferred target.

The OOM killer exists because the alternative is worse. Without it, the system would deadlock (no memory to allocate, no way to free any). At least killing one process gives everything else a chance to survive.

---

## why linux overcommits

By default, Linux allows you to allocate more memory than exists. `malloc(1GB)` succeeds even if you only have 512MB free.

This is **overcommit**, and it's intentional. Most programs allocate way more memory than they use. Sparse arrays, buffers "just in case," forked processes before exec. If Linux refused these allocations, lots of software would break.

The downside: if you actually try to use all that memory you allocated, you trigger the OOM killer. The allocation succeeded, but using it didn't.

You can tune this behavior:

```bash
# Check current mode
$ cat /proc/sys/vm/overcommit_memory
0  # heuristic (default)

# Options:
# 0 = heuristic overcommit (kernel guesses what's safe)
# 1 = always overcommit (never refuse malloc)
# 2 = strict (refuse if total committed memory would exceed a commit limit derived from RAM, swap, and vm.overcommit_ratio / vm.overcommit_kbytes)
```

Mode 2 is safer but breaks more software. Mode 0 is a compromise. Mode 1 is living dangerously.

---

## what this means practically

**Mysterious crashes.** If your process dies with no explanation, check `dmesg` for OOM killer messages. It might not be your bug (Linux might have killed you to save the system).

**Server planning.** On production servers, monitor swap usage. If you're consistently in swap, you're consistently slow. If swap fills up, the OOM killer is coming.

**Memory limits.** Use cgroups to limit how much memory a process can use. Better to have one application fail cleanly than to have the OOM killer pick arbitrarily.

**Kernel memory matters.** If kernel memory grows unbounded (driver bug, leak in a module), you can't swap it out. Eventually, OOM. Unlike user-space leaks, you can't just kill the offending process.

---

## the reality

Linux's memory management is a set of tradeoffs, not solutions.

Swappiness lets you trade application responsiveness for cache performance. Overcommit lets you trade guaranteed allocation for flexibility. The OOM killer trades one process's life for system stability.

None of these are free. The system is designed to keep running as long as possible, even when resources are exhausted. But "running" under memory pressure doesn't mean "running well."

When you see the OOM killer fire, it's not a bug. It's the kernel doing exactly what it's designed to do: making a hard choice so you don't have to reboot.
