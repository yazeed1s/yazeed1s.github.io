+++
title = "mmap: Mapping Files into Memory"
date = 2024-11-08
description = "How mmap works, when to use it, and when not to."
[taxonomies]
tags = ["linux", "memory", "systems programming"]
+++

`mmap` maps a file (or anonymous memory) into your process's address space so you can access it through pointers instead of read/write syscalls. The kernel sets up page table entries pointing to the file's pages, and when you access a mapped region the data appears in memory through the standard page fault mechanism.

## how it works

When you call `mmap`, the kernel creates a virtual memory area (VMA) in your process but doesn't actually load any data yet. The first time you access a page in the mapped region, a page fault fires, the kernel loads that page from the file into the page cache (or allocates a zero page for anonymous mappings), updates the page table, and your access continues. Subsequent accesses to the same page hit the page cache directly, no fault needed.

```c
int fd = open("data.bin", O_RDONLY);
struct stat st;
fstat(fd, &st);

void *ptr = mmap(NULL, st.st_size, PROT_READ, MAP_PRIVATE, fd, 0);
// now ptr[0..st.st_size-1] is the file contents
```

The file is accessible through pointer arithmetic. No `read()` calls, no user-space buffers. The kernel handles paging transparently.

## flags that matter

`mmap` behavior changes significantly based on the flags you pass, and the two most important are the sharing flags: `MAP_PRIVATE` creates a copy-on-write mapping where writes go to a private copy and the original file is unchanged, while `MAP_SHARED` means writes go through to the actual file (other processes mapping the same file see your changes, and changes eventually get written back to disk).

`MAP_ANONYMOUS` creates a mapping not backed by any file, just zero-filled pages, and this is what malloc uses internally for large allocations. `MAP_FIXED` forces the mapping at a specific address (dangerous if you don't know what's there), and `MAP_POPULATE` pre-faults all pages at mmap time instead of waiting for access, which avoids page faults later but means the mmap call itself is slow.

Protection flags control access: `PROT_READ` for read-only, `PROT_WRITE` for writable, `PROT_EXEC` for executable (JIT compilers use this), and `PROT_NONE` for no access (useful for guard pages). You can change protections later with `mprotect`.

## anonymous mappings

When you `mmap` with `MAP_ANONYMOUS`, there's no file involved, you just get zero-filled memory. This is actually how glibc malloc works for large allocations: small allocations come from `sbrk` (the heap), but anything over a threshold (usually 128KB) gets its own anonymous mmap.

```c
// what malloc does internally for large allocs
void *p = mmap(NULL, size,
    PROT_READ | PROT_WRITE,
    MAP_PRIVATE | MAP_ANONYMOUS,
    -1, 0);
```

The advantage is that when you free a large allocation, the kernel can immediately reclaim the pages since the mapping is independent. With sbrk, the heap can only shrink from the end, so fragmentation keeps memory allocated.

## shared memory

`MAP_SHARED` with `MAP_ANONYMOUS` gives you shared memory between parent and child processes (across `fork`), and `MAP_SHARED` with a named file gives you shared memory between unrelated processes. The file acts as the backing store, and the kernel ensures coherence through the page cache.

For IPC, you can also use `shm_open` + `mmap` to create a named shared memory region without a real file on disk, which is how many high-performance IPC mechanisms work.

## msync and coherence

With `MAP_SHARED` file mappings, your writes eventually make it to disk, but "eventually" isn't a guarantee of when. If you need to ensure data has been written, you call `msync` which flushes dirty pages to the file.

```c
msync(ptr, length, MS_SYNC);  // blocks until written
msync(ptr, length, MS_ASYNC); // schedules write, returns immediately
```

This matters for databases and anything that needs durability. Without explicit msync, a crash might lose recent writes. With MAP_PRIVATE you don't need msync because writes never go to the file anyway.

## when mmap is good

File I/O without syscall overhead is the classic use case, and it works well when you're reading a large file sequentially or doing random access across a file that mostly fits in memory. There are no read/write syscalls per access, just page faults on cold pages, and the kernel manages caching automatically.

Memory-mapped I/O also shines for read-only shared data where multiple processes can map the same file and share the physical pages through the page cache, meaning ten processes mapping the same 1GB file don't use 10GB of RAM.

Other good uses include loading shared libraries (`.so` files are mmap'd into process address space), JIT compilation (`PROT_EXEC` on anonymous mappings), and inter-process communication through shared mappings.

## when mmap is bad

For databases, mmap is [usually the wrong choice](@/posts/mmap_databases.md). The OS can flush dirty pages whenever it wants, which breaks write-ahead logging. Page faults stall threads unpredictably, and there's no async fault interface. Error handling goes through SIGBUS instead of return codes. And under memory pressure, TLB shootdowns during eviction can collapse throughput.

Sequential writes to a new file are also better with `write()` because mmap requires you to know the file size upfront (or use `ftruncate` to grow it), and the page-fault-per-page overhead can be worse than buffered writes.

Small files aren't worth the mmap overhead either, since the setup cost (VMA creation, page table manipulation) is higher than just calling read() once.

## munmap and cleanup

When you're done with a mapping, call `munmap` to release it. For `MAP_SHARED` mappings, you should `msync` first if you care about durability.

```c
msync(ptr, length, MS_SYNC);
munmap(ptr, length);
close(fd);
```

If you just exit without munmap, the kernel cleans up all mappings on process exit (same as all resources), but explicit cleanup is good practice for long-running processes to avoid accumulating VMAs.

## notes

- `MAP_HUGETLB` for huge page mappings, reduces TLB pressure for large mappings
- `madvise()` hints to the kernel about access patterns: `MADV_SEQUENTIAL`, `MADV_RANDOM`, `MADV_DONTNEED`
- `MADV_DONTNEED` on anonymous memory zeros the pages, on file-backed it drops them from cache
- The 64-bit address space means you can mmap very large files without worrying about address space exhaustion
- `mmap` on `/dev/mem` gives access to physical memory (requires root, used for hardware access)
- On 32-bit systems, mmap was limited by the 3GB user-space address space
