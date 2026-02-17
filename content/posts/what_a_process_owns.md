+++
title = "What a Process Really Owns"
date = 2026-01-05
description = "Breaking down what resources a process actually has and what the kernel tracks for it."
[taxonomies]
tags = ["os", "processes", "linux"]
+++

I wanted to think through what a process actually "owns" because I realized I had a fuzzy picture of it. We throw around words like "process", "thread", "address space" but I wanted to be more concrete about what the kernel actually tracks.

## the short version

A process owns:

- Its own virtual address space
- File descriptors
- Signal handlers
- Resource limits
- A process ID and various metadata
- One or more threads of execution

When you fork(), the child gets copies of most of these. When you exec(), most of it gets replaced with fresh stuff for the new program.

## address space

This is the big one. Every process gets its own virtual address space. On a 64-bit system that's a huge range of addresses (48 bits usable on x86-64, so 256 TB of virtual space).

The address space contains:

- **Text segment**: the executable code, usually read-only
- **Data segment**: initialized global/static variables
- **BSS**: uninitialized globals (zeroed on startup)
- **Heap**: dynamic allocations (malloc, new)
- **Stack**: function call frames, local variables
- **Memory-mapped regions**: shared libraries, mmap'd files, anonymous mappings

The kernel tracks all this in data structures (on Linux, the `mm_struct` and a tree of `vm_area_struct`). Each region has permissions (read/write/execute) and backing (file, anonymous, shared, private).

You can see your process's memory map in `/proc/self/maps`:

```bash
$ cat /proc/self/maps
55a4c8a00000-55a4c8a02000 r--p 00000000 08:01 1234567  /usr/bin/cat
55a4c8a02000-55a4c8a06000 r-xp 00002000 08:01 1234567  /usr/bin/cat
55a4c8a06000-55a4c8a09000 r--p 00006000 08:01 1234567  /usr/bin/cat
...
7f8c12000000-7f8c12021000 rw-p 00000000 00:00 0
7ffd5c9e0000-7ffd5ca01000 rw-p 00000000 00:00 0        [stack]
```

Each line is a region with its address range, permissions, offset, device, inode, and path.

## file descriptors

A process has a table of open file descriptors. These are just small integers (0, 1, 2, 3, ...) that refer to open files, pipes, sockets, devices, whatever.

By convention:

- 0 = stdin
- 1 = stdout
- 2 = stderr

When you open() a file, you get the lowest available fd number. When you fork(), the child inherits copies of the parent's fd table (pointing to the same underlying file objects, so they share the file offset).

The kernel tracks this in a `files_struct`. Each fd points to a `file` object which points to an inode.

You can see a process's fds in `/proc/[pid]/fd/`:

```bash
$ ls -la /proc/self/fd
lrwx------ 1 user user 64 Jan  1 00:00 0 -> /dev/pts/0
lrwx------ 1 user user 64 Jan  1 00:00 1 -> /dev/pts/0
lrwx------ 1 user user 64 Jan  1 00:00 2 -> /dev/pts/0
lr-x------ 1 user user 64 Jan  1 00:00 3 -> /proc/12345/fd
```

## credentials and IDs

A process has:

- **PID**: unique process ID
- **PPID**: parent process ID
- **UID/GID**: user and group IDs (real, effective, saved)
- **Process group and session IDs**: for job control

The UID/GID stuff is more complicated than I expected. There's the real UID (who you actually are), effective UID (what permissions you're currently using), and saved UID (so you can drop and regain privileges). This is how setuid programs work.

## signal handling

A process has a table of signal handlers. Each signal (SIGINT, SIGTERM, SIGSEGV, etc.) can have:

- Default action (terminate, ignore, stop, etc.)
- A custom handler function
- Ignored

Plus there's a signal mask (which signals are currently blocked) and pending signals (delivered but not yet handled).

## resource limits

Every process has limits on things like:

- Maximum file size it can create
- Number of open files
- Stack size
- CPU time
- Memory

You can see these with `ulimit -a` in bash. The kernel enforces these.

## threads

Here's where it gets a bit confusing. On Linux, threads are really just processes that share stuff. When you create a thread (via clone() with the right flags), the new "thread" shares the address space, file descriptors, signal handlers, etc. with the parent.

So a "process" might have multiple threads, and they all share most of the resources I listed above. But each thread has its own:

- Thread ID
- Stack
- Register state
- Signal mask
- errno
- Thread-local storage

The kernel calls these "tasks" internally. A process is really a group of tasks that share an address space.

## PCB and TCB

So all this stuff I've been talking about, the kernel has to store it somewhere. That's where the Process Control Block (PCB) and Thread Control Block (TCB) come in.

**Process Control Block (PCB)** is a kernel data structure that holds everything about a process. On Linux this is the `task_struct` (confusingly named since it's used for both processes and threads). It contains:

- Process state (running, sleeping, stopped, zombie)
- PID, PPID, credentials
- Pointers to the memory management structures (`mm_struct`)
- Pointer to the file descriptor table (`files_struct`)
- Signal handling info
- Scheduling info (priority, time slice, CPU affinity)
- Accounting info (CPU time used, etc.)
- Pointers to parent, children, siblings in the process tree

When the scheduler picks a process to run, it reads from the PCB to restore the process's context. When the kernel needs to check permissions or deliver a signal, it looks at the PCB.

**Thread Control Block (TCB)** stores the execution context for a specific thread. On Linux this is also in `task_struct` since threads and processes use the same structure. But the thread-specific stuff includes:

- Register state (saved during context switch)
- Stack pointer
- Thread ID
- Signal mask (which signals this thread blocks)
- Thread-local storage pointer
- errno location

The key difference is that multiple TCBs (threads) can point to the same memory management structures and file descriptor tables, while separate PCBs (processes) have their own.

When you do a context switch, the kernel saves the current thread's register state into its TCB, then loads the next thread's state from its TCB. If the threads belong to different processes, it also has to switch page tables (which is the expensive part).

You can think of it like this: the PCB holds "what resources does this process own" while the TCB holds "where is this thread in its execution". A single-threaded process has one PCB and effectively one TCB. A multi-threaded process has one PCB (shared resources) and multiple TCBs (one per thread).

## what happens on fork()

When you fork():

- Address space is copied (or copy-on-write, so it's cheap until you modify things)
- File descriptors are copied (but point to the same file objects)
- Signal handlers are copied
- The child gets a new PID

The child is almost identical to the parent at the moment of fork. This is why fork+exec is the traditional Unix way to spawn programs: fork copies everything, then exec replaces it all with a new program.

## what I find interesting

The address space isolation is the core thing. That's what makes a process a process. Everything else (fds, credentials, signals) is just bookkeeping.

Threads share the address space but are otherwise separate schedulable entities. This is why data races are possible with threads but not between processes (unless you explicitly share memory).

The /proc filesystem is amazing for introspection. You can see almost everything about a running process without any special tools. Just cat files.

## notes

- On Linux, look at `task_struct` in the kernel source to see what the kernel tracks per task
- `mm_struct` holds the memory management info
- `files_struct` holds the fd table
- `signal_struct` and `sighand_struct` handle signals
- The clone() syscall lets you pick exactly what to share between parent and child, which is how threads are implemented
- cgroups and namespaces add more layers of isolation on top of this (for containers)

If you want to poke around a running process, check `/proc/[pid]/status` (state, memory, threads), `/proc/[pid]/maps` (memory map), `/proc/[pid]/fd` (open file descriptors), `/proc/[pid]/limits` (resource limits), `/proc/[pid]/task/` (threads). Also `ulimit -a` for your shell's limits and `grep ctxt /proc/self/status` for context switch counts.

