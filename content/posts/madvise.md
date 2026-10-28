+++
title = "madvise, Or Talking Back to the Kernel"
date = 2026-10-28
description = "The small syscall where user space gets to have an opinion about its own memory."
[taxonomies]
tags = ["OS", "memory", "linux"]
+++

In the [OPT post](@/posts/opt.md) I ended up at `madvise()` as the thing that already exists for "telling the kernel about the future", and I kept reading about it after, because it's kinda the only place in Linux where user space is officially allowed to have an opinion about its own pages. Everything else about virtual memory is the kernel deciding and you not knowing.

```c
int madvise(void *addr, size_t length, int advice);
```

You give it a page-aligned range and a flag, and the kernel does something, or nothing, depends on the flag. That "or nothing" part is what makes it interesting to me.

## hints that are actually hints

`MADV_SEQUENTIAL` and `MADV_RANDOM` describe how you're going to touch a file-backed mapping, so the kernel reads ahead more aggressively for sequential and basically turns readahead off for random. `MADV_WILLNEED` says I'll need this range soon, and for a file mapping the kernel starts reading it in asynchronously so the fault later is cheap (or doesn't happen). These are the polite ones, the kernel can ignore them and your program is still correct, just maybe slower.

`MADV_HUGEPAGE` is in the same family. If THP is set to `madvise` mode (common on servers), only ranges marked with this get huge pages, so it's an opt-in, and `MADV_NOHUGEPAGE` is the opt-out for the cases where huge pages hurt you, which happens more than people expect.

## and one that is not a hint at all

`MADV_DONTNEED` sounds like the polite "I don't need this anymore, reclaim it if you want", but on private anonymous memory Linux throws the pages away right now, and the next access gets a fresh zero page.

```c
size_t len = 64 << 20;
char *p = mmap(NULL, len, PROT_READ | PROT_WRITE,
               MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
memset(p, 'a', len);

madvise(p, len, MADV_DONTNEED);
printf("%d\n", p[0]);   // 0, the 'a's are gone
```

So it's a free that keeps the mapping, which is actually useful (allocators use it exactly for this), but it's a command with a hint's name. POSIX has `posix_madvise(POSIX_MADV_DONTNEED)` which is supposed to be a real hint, and glibc on Linux just makes it a no-op, I'd guess because mapping it to the Linux one would silently destroy data in programs written for the POSIX meaning. lol.

## MADV_FREE and the RSS confusion

`MADV_FREE` (4.5) is the lazier version. It tells the kernel these pages are garbage, but it doesn't drop them until there's memory pressure, and if you write to the page before that happens the kernel just cancels the free and keeps it. For an allocator this is nicer, because a lot of freed memory gets reused soon and with `DONTNEED` you'd pay a fault plus zeroing every time.

The problem is RSS doesn't go down. The pages are still counted against you until the kernel actually reclaims them, so tools and people looking at `top` think the program is leaking. Go switched its runtime to `MADV_FREE` and then switched back to `MADV_DONTNEED` by default a few versions later, mostly because of this, people saw memory "not being returned" and the monitoring and OOM decisions around containers believed RSS. I find that kinda funny, but I get it, if every dashboard shows the process growing nobody cares that the memory is technically free.

## COLD and PAGEOUT

`MADV_COLD` and `MADV_PAGEOUT` came in 5.4 and these are closest to the OPT thing. `COLD` moves the pages to the inactive list so they're first in line for reclaim, while `PAGEOUT` reclaims them now, anon pages go to swap and clean file pages get dropped. Neither destroys data, the next access just faults it back in.

These came from Android, where the system knows an app went to the background long before the kernel's LRU figures it out. Which is the "the app knows more than the kernel" argument again, except the knowledge comes from the system watching the app, and the app itself doesn't say anything. And for that there's `process_madvise()` (5.10), which lets one process give advice about another process's memory through a pidfd. Only a few flags are allowed there (`COLD`, `PAGEOUT`, `WILLNEED`, `COLLAPSE` I think) and you need `CAP_SYS_NICE` plus ptrace-level access to the target, which makes sense, letting random processes push other people's memory to swap would be a fun attack.

This also kinda answers the trust question I had in the OPT post. Plain `madvise` only touches your own memory, so the worst a bad hint can do is hurt you, and the kernel doesn't need to trust you at all. The moment advice crosses a process boundary it needs permissions. I'm not sure that's a designed principle or it just happened that way, but it works.

## the weird ones

Some flags aren't about caching at all and feel like they ended up in `madvise` because it was the syscall that takes a range. `MADV_DONTFORK` keeps a range out of the child after `fork()` (RDMA buffers need this), `MADV_WIPEONFORK` gives the child zeros instead of a copy, which is nice for RNG state or keys you don't want two processes to share by accident, and `MADV_DONTDUMP` keeps a range out of core dumps. `MADV_MERGEABLE` opts the range into KSM. `MADV_POPULATE_READ` / `WRITE` prefault the range so you don't take the faults later, which is basically `MAP_POPULATE` but for an existing mapping.

At this point it's less "advice" and more a drawer for per-range memory flags. I don't think that's bad, but the name stopped meaning much a while ago.

## what I think about it

The thing I like is how small the leak in the abstraction is. The default is still the kernel deciding everything, and most programs never call `madvise` and that's fine. The programs that do call it are mostly allocators, runtimes, databases and Android, people who already know their memory better than the kernel and want a way to say so without taking over (which is what `O_DIRECT` and buffer pools do).

What's missing for me is anything with a notion of time, all of this is "now" or "soon" or "not anymore", nothing like "I'll need this in 10 seconds". Would the kernel even do anything useful with that? no idea, maybe not.

## notes

- `addr` has to be page-aligned, length gets rounded up to pages.
- `MADV_DONTNEED` on shared or file-backed mappings doesn't zero anything, the next access just reads from the file/page cache again. The zeroing is the private anonymous case.
- `MADV_FREE` only works on private anonymous memory.
- `MADV_COLLAPSE` (6.1) synchronously collapses a range into huge pages instead of waiting for khugepaged.
- `MADV_REMOVE` frees the backing store for shmem/tmpfs ranges, it punches a hole.
- `man 2 madvise` is honestly pretty readable for once.
