+++
title = "Context Switches: What Actually Happens"
date = 2024-11-18
description = "A context switch is the mechanism that lets the OS multitask."
[taxonomies]
tags = ["OS", "CPU", "Linux"]
+++

A context switch is what happens when the OS takes one process off the CPU and puts another one on. It happens constantly, thousands of times per second, and your programs never notice because the whole point is that it looks seamless.

## why context switches exist

CPUs don't multitask on their own, they execute one instruction stream at a time (per core). The OS creates the illusion of parallelism by rapidly switching between processes, giving each one a time slice of maybe 1-10 milliseconds, executing its instructions, then saving its state and loading the next process's state. Do this fast enough and it looks like everything runs at the same time.

## what gets saved

A process's execution state lives in CPU registers, and when the OS switches away from a process it needs to save all of them: general purpose registers (rax, rbx, rcx on x86-64), the instruction pointer (rip, which says where execution was), the stack pointer (rsp), flags register, floating point and SIMD registers (SSE, AVX), and the memory mappings reference (CR3 on x86, which points to the page tables).

All of this gets saved to the process's kernel data structure (the task_struct on Linux), and the incoming process's saved state gets loaded into the CPU registers. The CPU then continues executing from wherever the new process left off.

## what actually happens step by step

The timer interrupt fires (or the process yields, or it blocks on I/O), the CPU traps to the kernel, the scheduler picks the next process, the kernel saves current registers to the outgoing task_struct, loads registers from the incoming task_struct, switches the page tables by writing CR3 (which changes the entire virtual memory mapping), flushes TLB entries that are no longer valid, and returns to user space where the new process resumes as if nothing happened.

```
Process A running
  -> timer interrupt fires
  -> save A's registers to A's task_struct
  -> scheduler picks Process B
  -> load B's registers from B's task_struct
  -> switch page tables (CR3)
  -> flush TLB
  -> Process B is now running
```

The key thing is that the process doesn't know. It saved no state, it called no function. The kernel did everything while the process wasn't looking.

## the cost

Context switches aren't free. The direct cost is saving and restoring register state, which is maybe a few hundred nanoseconds. But the indirect cost is worse: the TLB gets flushed (partially or fully) because the new process has different page tables, so the first memory accesses after the switch take page table walks instead of TLB hits. The CPU caches (L1, L2) are now full of the old process's data, and the new process suffers cache misses until it warms them up. Branch predictors trained on the old process's code are useless for the new process.

These indirect costs can add up to several microseconds of effective penalty, and on workloads with many short-lived operations it can matter a lot.

## thread switches vs process switches

Threads within the same process share address space, so switching between them doesn't require changing CR3 or flushing the TLB. That makes thread switches cheaper: you still save/restore registers, but you skip the expensive page table swap and TLB invalidation.

This is one reason why multi-threaded servers outperform multi-process ones for high-concurrency workloads, fewer and cheaper context switches.

## voluntary vs involuntary

A **voluntary** context switch happens when a process can't continue: it calls `read()` and waits for disk, calls `sleep()`, waits on a mutex, or does any blocking operation. The process is saying "I have nothing to do, give the CPU to someone else."

An **involuntary** context switch happens when the scheduler preempts the process because its time slice expired (the timer interrupt fires and the scheduler decides it's someone else's turn), a higher-priority process becomes runnable, or load balancing moves the process to another core. You can see both types in `/proc/<pid>/status` under `voluntary_ctxt_switches` and `nonvoluntary_ctxt_switches`.

## what triggers a switch

Most context switches come from I/O waits (process blocks, voluntary), timer expiry (time slice used up, involuntary), synchronization (mutex, semaphore, condition variable), and inter-process communication (pipe, signal). A busy process that never blocks still gets switched out involuntarily when its time slice expires.

## notes

- `perf stat` shows context switch counts for a command
- `/proc/<pid>/status` shows voluntary and involuntary counts per process
- `vmstat` shows system-wide context switches per second (cs column)
- On a typical desktop system you might see 10,000-50,000 context switches per second
- PCID (Process Context ID) on modern x86 lets the TLB keep entries from multiple processes, reducing the flush cost
- Kernel preemption means the kernel itself can be context-switched mid-operation (when configured with `PREEMPT`)
