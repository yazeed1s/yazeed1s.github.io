+++
title = "RDMA: Remote Direct Memory Access"
date = 2025-03-23
description = "How RDMA enables low-latency, zero-copy network communication."
[taxonomies]
tags = ["RDMA", "Networking", "InfiniBand", "High Performance Computing", "Hardware"]
+++

---

## Introduction

- What is RDMA?
- Why it matters (low latency, high throughput, zero CPU copy)
- Where it's used (HPC, data centers, storage)

---

## Traditional Networking vs RDMA

- TCP/IP stack overhead
- Kernel bypass
- Zero-copy transfers
- CPU involvement comparison

---

## RDMA Technologies

- InfiniBand
- RoCE (RDMA over Converged Ethernet)
- iWARP
- Comparison table

---

## Key Concepts

- Queue Pairs (QP)
- Completion Queues (CQ)
- Memory Regions (MR)
- Protection Domains (PD)
- Verbs API

---

## RDMA Operations

- Send/Receive
- RDMA Read
- RDMA Write
- Atomic operations

---

## Programming RDMA

- libibverbs basics
- Simple example: connection setup
- One-sided vs two-sided operations

---

## Use Cases

- Distributed storage (Ceph, SPDK)
- Databases
- Machine learning training
- Memory disaggregation (link to memory-disaggregation.md)

---

## Challenges

- Complex programming model
- Hardware requirements
- Debugging difficulties

---

## Conclusion

- When to use RDMA
- Future: CXL vs RDMA (link to cxl.md)

---

## References

- RDMA programming guides
- Mellanox/NVIDIA documentation
