+++
title = "OPT, and telling the OS about the future"
date = 2024-10-01
description = "Would breaking OS transparency solve this?"
[taxonomies]
tags = ["OS", "cache", "memory"]
+++

In one of the lectures the instructor was going through cache replacement policies, LRU, FIFO, clock, the usual ones, and then he mentioned OPT, which is the one that is actually optimal and also the one nobody can implement. I wrote "blame OS transparency?" next to it in my notes and it stayed there for sometime, so this is me finally looking at it.

## what OPT does

When the cache is full and you need a slot, OPT evicts the page whose next use is furthest in the future, and a page that is never used again is the perfect victim. Belady showed in 1966 that this gives the minimum number of misses (that's why it's also called MIN, or Belady's algorithm, same thing). The problem is the kernel would need the future access sequence and it obviously doesn't have it, so OPT ends up as a measuring stick where you record a trace, run OPT on it offline, and see how far LRU or clock is from it.

## my lazy thought

The kernel only sees page faults and accessed bits, it has no idea my program is about to scan a 10GB file once and never touch it again, or that some table will be hit again in two seconds. But the program kinda knows that. So why can't it just tell the kernel something like "this page, I'll need it much later", or even hand over the path the code is going to take and let the kernel do something closer to OPT?

I thought I was onto something, and then found out people did this a long time ago.

## it already exists, sort of

Linux has `madvise()` for memory and `posix_fadvise()` for files, and they're basically this idea in a very coarse form. `MADV_DONTNEED` / `POSIX_FADV_DONTNEED` say I'm done with this, `MADV_WILLNEED` says I'll need it soon so go prefetch, `MADV_SEQUENTIAL` and `MADV_RANDOM` describe the pattern, and newer kernels (5.4 I think) added `MADV_COLD` and `MADV_PAGEOUT` which mean "this is cold, deactivate it" and "just reclaim it now". `POSIX_FADV_NOREUSE` was a no-op on linux for years, I think it got wired up to something recently but I haven't checked how much it actually does. And `mlock()` goes the other direction and pins pages so they never get evicted.

None of these say "next use is at time t", they're more like near vs far, but ranking near vs far is most of what OPT needs anyway, if the hints are honest that is.

Databases skipped the conversation completely, they open files with `O_DIRECT` and run their own buffer pool because they have the query plan, and a query plan is a pretty good description of the future.

On the research side this was a 90s thing. Cao, Felten and Li had application-controlled file caching where the kernel decides how much cache each process gets and the process decides which of its own blocks to drop, and Patterson et al. had informed prefetching and caching where the app discloses its future accesses to the kernel, which is literally the "give the OS the path" line from my note lol. Exokernel went all the way and said the OS shouldn't hide resources at all, just multiplex them safely and let apps manage, so breaking transparency was not exactly a new idea, MIT built a whole OS around it.

## why it's still not OPT

The page cache is shared, so even if my program tells the kernel its exact future, global OPT needs everyone's future and the other processes aren't saying anything. The 90s work handles this by splitting it, the kernel does allocation between processes and each app does replacement inside its own share, which is reasonable, but you only get something close to OPT inside your own slice.

There's also trust, because if hints affect eviction then every app will say its pages are important. I'd guess that's part of why `mlock` is capped by `RLIMIT_MEMLOCK`. Hints that evict your own stuff sooner are safe to honor, hints that push other people's stuff out are not.

And the program often doesn't know the future either, which ngl kills most of my original idea. A scan or a loop over an array is predictable, but anything that branches on data, or waits on user input or the network, has a path that depends on values you don't have yet. You could hand the kernel probabilities instead, but then you're predicting again, which is what LRU already does.

One more thing that bugged me, OPT minimizes misses and not cost. Evicting a dirty page means a write while evicting a clean one is free, and when objects have different sizes (CDNs) the whole problem changes, so even perfect knowledge of the future doesn't give you the best policy in the sense you actually care about.

## where OPT actually runs

It runs wherever the future is known. Register allocation is one, when the compiler has to spill inside a basic block it can pick the value whose next use is furthest because it sees all the code, and that's Belady again with registers instead of pages. DNN training is another, the compute graph is fixed so you know exactly when each tensor is needed, and systems that swap tensors between GPU and host memory plan around that.

Then there's learning from OPT, which I like more than the hint stuff. Hawkeye (CPU caches) runs OPT over past accesses and trains a predictor to guess whether OPT would have kept a new line, and LRB does something similar for CDNs. You can't know the future, but you can look at the past, ask what OPT would have done, and hope the future looks similar enough.

So would breaking transparency solve it? partially, for programs that really know their access pattern, and those mostly already took the cache away from the kernel. For general stuff I don't think it helps much, because the app usually doesn't know its future either. Not sure what a hint for "data-dependent future" would even look like.

## notes

- Belady, "A study of replacement algorithms for a virtual-storage computer", IBM Systems Journal, 1966.
- Cao, Felten, Li, application-controlled file caching, USENIX 1994.
- Patterson et al., "Informed Prefetching and Caching", SOSP 1995.
- Engler, Kaashoek, O'Toole, "Exokernel", SOSP 1995.
- Hawkeye is Jain & Lin, ISCA 2016. LRB is Song et al., NSDI 2020.
- `madvise` hints are hints, the kernel is free to ignore most of them.
