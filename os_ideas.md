1. eBPF-based real-time RTMP/SRT traffic inspector — Build an eBPF program that hooks into the socket layer to track per-stream bitrate, packet loss, and jitter without userspace polling overhead. Compare overhead vs. traditional tcpdump-based monitoring.

2. Filesystem for immutable video segments — Design (or extend an existing FS like FUSE) a purpose-built filesystem optimized for write-once, read-many small segment files, comparing metadata overhead against ext4/XFS at scale (millions of small segment files).


3. hugepage-aware malloc

4. reimplemnting malloc, either jemalloc, ptmalloc, or dlmalloc and understanding them deeply and what issue they each solve. The goal is to have a deeper understanding of memory allocators and the challenges in that space