+++
title = "Context Switches"
date = 2025-04-03
description = "What context switches are, why we need them, and what they cost."
[taxonomies]
tags = ["OS", "CPU", "Scheduling"]
+++

A CPU can only run one thing at a time. When the OS needs to switch from one process to another, it saves everything about the current one and loads everything about the next. That's a context switch.

## what gets saved

"Everything" means the program counter (where the process was in its code), the register values, the stack pointer, and some other CPU state. This is called the process context.

The reason we need this is simple: we have more processes than CPUs. On my laptop I might have hundreds of processes but only 8 cores. They all need to run somehow, so they take turns. The OS gives each process a slice of time on a CPU, and when the slice is up (or something else happens), it switches to another process.

## when do context switches happen

A few situations:

**Timer interrupt.** The scheduler sets a timer (maybe 1-10ms depending on the OS and config). When it fires, the kernel gets control and can decide to switch to another process. This is called preemption.

**Process blocks.** If a process does something that has to wait (reading from disk, waiting for network, waiting for a lock), it makes no sense to keep it on the CPU. The kernel switches to something that can actually run.

**Explicit yield.** A process can voluntarily give up the CPU. This is rare in practice, most of the time the kernel just preempts.

**Higher priority process wakes up.** If something more important becomes runnable (like a real-time task or an interactive process that was waiting for input), the kernel might preempt the current process immediately.

## what actually happens during a switch

When the kernel decides to switch from process A to process B:

1. Save A's registers to memory (usually in some kernel data structure associated with A, like the task_struct in Linux)
2. Save A's stack pointer
3. Update page tables or TLB if the processes have different address spaces (this is where it gets expensive)
4. Load B's registers from memory
5. Load B's stack pointer
6. Jump to wherever B left off

If A and B are threads in the same process, step 3 is simpler because they share the same address space. This is one reason thread switches are cheaper than process switches.

> This is the common case, but not always true. With Spectre mitigations like KPTI enabled, the kernel uses separate page tables for user and kernel space. Even a thread switch within the same process pays for the kernel page table switch on entry and exit.

## why context switches are expensive

The direct cost isn't that bad. Saving and restoring registers takes maybe hundreds of nanoseconds to a few microseconds. The bigger costs are indirect:

**TLB flush.** The TLB caches virtual-to-physical address translations. If you switch to a process with a different address space, those cached translations are useless (or worse, wrong). The TLB gets flushed and the new process starts cold. Every memory access causes a TLB miss until the cache warms up again.

**Cache pollution.** The new process touches different memory. The L1/L2/L3 caches fill up with its data, evicting the old process's data. When you switch back, you start with cold caches again.

**Pipeline flush.** Modern CPUs have deep pipelines with speculative execution. A context switch might flush all of that.

So the direct cost is maybe 1-10 microseconds depending on the hardware. But the indirect costs from cache/TLB warming can add hundreds of microseconds or more to the next stretch of execution.

## why this matters

If you're doing I/O-bound work, context switches are probably fine. You're waiting on disk or network anyway, the overhead is noise compared to the wait time.

If you're doing CPU-bound work and switching a lot, you're wasting cycles. This is why things like busy-waiting or spinning on a lock can sometimes outperform blocking (at least for short waits). You avoid the context switch overhead.

It's also why async I/O and event loops (like epoll, io_uring, kqueue) are popular. Instead of blocking and switching, you check if something is ready and move on. You stay on the CPU. Fewer switches, warmer caches.

## some numbers

These are rough and depend heavily on hardware and workload:

- Direct context switch cost: ~1-5 μs on modern hardware
- TLB flush + warmup: can add 10-100+ μs depending on working set size
- Thread switch (same process): cheaper, maybe 0.5-2 μs since no page table switch
- Typical scheduler timeslice: 1-10 ms

The kernel tries to be smart about this. Linux's CFS scheduler considers cache locality when picking where to run a task. It tries to keep tasks on the same CPU they ran on last time (CPU affinity) to preserve cache warmth.

## notes

- You can measure context switches on Linux with `perf stat -e context-switches ./your_program` or by reading `/proc/[pid]/status`
- `vmstat 1` shows system-wide context switches per second
- Voluntary vs involuntary: voluntary means the process blocked or yielded, involuntary means the scheduler preempted it
- In user-space threading (green threads, goroutines), "context switches" are much cheaper because you're not going through the kernel and not touching page tables
