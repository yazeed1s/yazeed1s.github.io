+++
title = "Infiniswap: Remote Memory Paging Over RDMA"
date = 2025-03-23
description = "How Infiniswap uses RDMA to swap pages to remote memory instead of disk."
[taxonomies]
tags = ["Memory", "RDMA", "Paging", "Distributed Systems", "Linux"]
+++

---

## Introduction

- What is Infiniswap?
- The paper: "Infiniswap: Efficient Memory Disaggregation" (NSDI 2017)
- Core idea: swap to remote memory, not disk

---

## Background

- Traditional swapping (link to swap-paging.md)
- Why disk swap is slow
- RDMA basics (link to rdma.md)

---

## How Infiniswap Works

- Architecture overview
- Page fault handling
- Remote memory servers (daemons)
- Block device interface

---

## Key Design Decisions

- Transparency (works with unmodified applications)
- Integration with Linux swap subsystem
- Load balancing across remote memory servers

---

## Performance

- Latency comparison: disk vs remote memory
- Throughput numbers from the paper
- Real workload results

---

## Limitations

- Network dependency
- Memory server failures
- Not a replacement for local memory

---

## Related Work

- Other remote paging systems
- Comparison with Fastswap, LegoOS, etc.

---

## Conclusion

- When to use Infiniswap
- Impact on memory disaggregation research

---

## References

- Infiniswap paper
- Source code repository
