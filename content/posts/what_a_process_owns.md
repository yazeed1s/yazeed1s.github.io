+++
title = "What a Process Owns"
date = 2025-10-19
description = "Everything a process has: address space, file descriptors, signals, credentials, and scheduling context."
[taxonomies]
tags = ["OS", "Linux", "Processes"]
+++

A process isn't just "running code." It's a container that owns a set of resources the kernel manages on its behalf, and understanding what those resources are helps explain why certain operations behave the way they do.

## address space

Each process gets its own virtual address space, which is a complete illusion of having memory all to itself. On 64-bit Linux a process gets a 128TB virtual space (48-bit addressing), split into user space (lower half) and kernel space (upper half, shared across all processes but inaccessible from user mode).

Inside this space you have the text segment (your compiled code, read-only and executable), the data segment for initialized globals, BSS for uninitialized globals (zeroed by the kernel), the heap growing upward from the end of BSS (managed by malloc/brk/mmap), the stack growing downward from near the top of user space, and memory-mapped regions scattered in between for shared libraries, mmap'd files, and anonymous mappings.

Two processes can have the same virtual address pointing to completely different physical memory because each process has its own page table. The kernel switches page tables on every context switch by writing a new value to CR3 on x86.

You can look at a process's memory map through `/proc/<pid>/maps`, which shows each VMA (Virtual Memory Area) with its address range, permissions, offset, device, inode, and pathname.

## file descriptors

A file descriptor is an integer index into a per-process table of open files. When you call `open()`, the kernel creates an entry pointing to the underlying file object and returns the lowest available integer, which is why `stdin` is 0, `stdout` is 1, and `stderr` is 2 (they're opened first).

Each process has its own file descriptor table, so fd 5 in one process is completely unrelated to fd 5 in another. The file descriptor carries the current read/write position, flags (like `O_NONBLOCK`), and a reference to the underlying inode.

File descriptors aren't just for files on disk: they represent pipes, sockets, eventfd, timerfd, signalfd, epoll instances, and even directories. The "everything is a file" philosophy means one interface for many types of I/O.

There's a per-process limit on open file descriptors (check with `ulimit -n`, default often 1024) and a system-wide limit. Leaking file descriptors is a common bug, especially in long-running services that open connections and forget to close them.

You can see all open file descriptors for a process in `/proc/<pid>/fd/`, where each entry is a symlink to the actual file, socket, or pipe.

## signals

Each process has a signal disposition table describing what happens for each signal. The default action depends on the signal: `SIGTERM` terminates, `SIGSTOP` suspends, `SIGCHLD` is ignored by default. You can override most defaults with signal handlers, except for `SIGKILL` and `SIGSTOP` which can't be caught or ignored.

A process also has a signal mask that blocks specific signals from being delivered (they queue up until unblocked), and a set of pending signals that have been sent but not yet handled.

Signal handling is per-process (signal disposition) but delivery can target specific threads. `kill(pid, sig)` sends to the process, `pthread_kill(tid, sig)` targets a thread.

## credentials

Every process has a set of UIDs and GIDs that determine what it can access. There are three sets: **real** (who actually started the process), **effective** (what's checked for access), and **saved** (the previous effective, so you can drop and regain privileges).

Usually real and effective are the same, but setuid programs differ. When you run `passwd`, the file has the setuid bit set, so the effective UID becomes root (owner of the file) while the real UID stays yours. The process can access `/etc/shadow` because effective UID is root, but the program knows who actually called it through the real UID.

Groups work the same way, and there's also a supplementary group list for all the groups you belong to. All of this is stored in the process's credential structure in the kernel.

## scheduling context

The kernel tracks scheduling metadata for each process: its priority (nice value from -20 to 19, and real-time priority if applicable), which scheduling class it belongs to (CFS for normal processes, FIFO or round-robin for real-time), how much CPU time it has consumed, when it last ran, and which CPU it ran on.

CFS (Completely Fair Scheduler) on Linux uses a virtual runtime to track how much CPU time a process has received relative to others, and it always picks the process with the lowest virtual runtime to run next. The nice value adjusts the weight: a lower nice value means more weight, which means the virtual runtime advances more slowly, which means more CPU time.

## namespaces and cgroups

In containerized environments, processes also own namespace memberships and cgroup associations. Namespaces give a process an isolated view of system resources: PID namespace makes it think its PID is 1, network namespace gives it a separate network stack, mount namespace gives it a different filesystem view.

Cgroups limit how much resource a process (or group of processes) can use: CPU quota, memory limit, I/O bandwidth, and number of PIDs. These are the building blocks of containers.

## the task_struct

Internally, the kernel represents all of this in a single structure called `task_struct`, which on Linux is a large struct (several kilobytes) containing or pointing to everything described above: the memory descriptor for address space, the files struct for file descriptors, the signal handler table, credentials, scheduling state, namespace pointers, cgroup references, and many other fields.

Every running thread has a `task_struct`. A single-threaded process has one. A multi-threaded process has one per thread, and they share the memory descriptor, file table, and signal handlers through pointers to the same structures.

## notes

- `/proc/<pid>/status` gives a summary of most of these: UIDs, GIDs, threads, memory usage, signal masks, context switch counts
- `/proc/<pid>/maps` for address space layout
- `/proc/<pid>/fd/` for open file descriptors
- `/proc/<pid>/cgroup` for cgroup membership
- `ls -la /proc/<pid>/ns/` shows namespace memberships
- `strace -p <pid>` lets you watch syscalls in real time
- A zombie process has exited but its parent hasn't called `wait()`, so the task_struct stays around holding the exit status
