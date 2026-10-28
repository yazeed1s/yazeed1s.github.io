+++
title = "mmap: Mapping Files into Memory"
date = 2024-11-08
description = "How mmap works, when to use it, and when not to."
[taxonomies]
tags = ["linux", "memory", "systems programming"]
+++

`mmap` is one of those calls where the API is tiny and what happens behind it is most of the virtual memory system. You ask for a file (or just zeroed memory) to show up in your address space, you get a pointer back, and from then on you read and write the file like an array without any `read()` calls.

```c
int fd = open("data.bin", O_RDONLY);
struct stat st;
fstat(fd, &st);

char *p = mmap(NULL, st.st_size, PROT_READ, MAP_PRIVATE, fd, 0);
// p[0..st.st_size-1] is the file now
```

## nothing is loaded

The part that took me a bit to accept is that `mmap` doesn't read anything. The kernel just creates a VMA (a note saying "this range of addresses belongs to that file at that offset") and returns, so the actual loading happens later when you touch a page, it faults, the kernel pulls that page into the page cache, points your page table at it, and you continue like nothing happened. Second touch is a normal memory access, no kernel involved at all in it, and that's why it can be faster than `read()`, there's no copy into a user buffer and no syscall per access.

## private vs shared

This is the flag that actually changes meaning. `MAP_PRIVATE` is copy-on-write, you read the file's pages but the first write gives you your own copy and the file never sees it. `MAP_SHARED` writes go into the page cache pages themselves, so other processes mapping the same file see them and they reach disk eventually, and "eventually" means whenever the kernel feels like it unless you call `msync`.

Add `MAP_ANONYMOUS` and there's no file at all, just zero pages. glibc malloc does this for big allocations (over 128KB by default), which is nice because `munmap` gives the memory straight back to the kernel, while memory from `brk` can only shrink from the top of the heap. Anonymous plus shared gives you memory that survives `fork` as shared between parent and child, and `shm_open` + `mmap` does the same for unrelated processes.

## where it's good and where it hurts

It's good for reading big files with random access, and for read-only stuff many processes use, since they all map the same page cache pages, ten processes mapping a 1GB file don't cost 10GB. Shared libraries are loaded this way, and JITs use anonymous mappings with `PROT_EXEC`.

It hurts when you need control. The kernel decides when dirty pages hit disk, a page fault blocks your thread and you can't make it async, and if the file shrinks under you or the disk throws an I/O error you get `SIGBUS` instead of an error code, which is kinda horrible to handle. For databases all of this adds up, enough that most of them stopped using it. And for small files I don't think it's worth it, setting up the VMA and taking faults probably costs more than one `read()`, though I never measured it.

Appending to a file is also awkward, the mapping has a fixed size so you `ftruncate` to grow the file and remap, and at that point `write()` is just simpler.

## notes

- `MAP_POPULATE` prefaults everything at map time, so `mmap` gets slow but later accesses don't fault.
- `MAP_FIXED` puts the mapping at your exact address and silently replaces whatever was there. `MAP_FIXED_NOREPLACE` is the safer one.
- `PROT_NONE` pages are good as guard pages, `mprotect` changes protection later.
- `MAP_HUGETLB` for huge pages, less TLB pressure.
- `madvise()` tells the kernel how you'll access the range (`MADV_SEQUENTIAL`, `MADV_RANDOM`, `MADV_WILLNEED`).
- Exiting without `munmap` is fine, the kernel cleans up. Long-running stuff should unmap tho, or VMAs pile up.
