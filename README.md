# DGX Spark Duo: Two-Node Cluster Guide

A step-by-step guide to connecting two NVIDIA DGX Spark (GB10) nodes into a
working two-node cluster, from first boot to multi-node inference.

## What this covers

| # | Section | What you get |
|---|---|---|
| 01 | [OS bring-up](01-os-bringup.md) | Hostnames, time sync, operator user, passwordless SSH, current firmware |
| 02 | [RoCE data-fabric networking](02-networking.md) | Direct-connect QSFP link up at MTU 9000, verified raw RDMA |
| 03 | [NCCL](03-nccl.md) | NCCL + nccl-tests built for Blackwell, allgather/allreduce at the ceiling |
| 04 | [Ray](04-ray.md) | Two-node Ray cluster, 2 nodes / 2 GPUs visible |
| 05 | [Docker + NVIDIA Container Toolkit](05-docker.md) | Containers can see the Blackwell GPU |
| 06 | [vLLM](06-vllm.md) | One model served across both GPUs with tensor parallelism |
| 07 | [Acceptance + troubleshooting](07-acceptance-troubleshooting.md) | End-to-end checklist plus the common failure modes |

Read them in order. Each section lists its prerequisites and ends with the
criteria that prove it is done; section 07 compresses all of them into one
pass/fail checklist.

## The two planes, kept apart

The whole guide rests on keeping two networks separate:

- **Management plane** is Tailscale (out-of-band). SSH, admin, Ray's control
  traffic, and the vLLM HTTP endpoint all live here.
- **Data plane** is the QSFP RoCE direct-connect link between the two
  ConnectX-7 NICs. Only RDMA traffic (NCCL collectives and vLLM tensor-parallel
  allreduce) uses it.

Do not mix them. The direct-connect subnets have no gateway, no DNS, and no
route off the box.

## Hardware reality (read this before believing any number)

- The ConnectX-7 NIC is rated **200 Gbps** (about 25 GB/s per port), not
  200 GB/s. Two NICs per node, each on PCIe 5.0 x4.
- **273 GB/s** is the LPDDR5x unified-memory bandwidth, not network bandwidth.
- The two NICs are bridged into a single L2 domain by the on-die eSwitch, so
  one QSFP cable lights up both port-1 interfaces. That is expected.
- **MTU 9000 is mandatory.** At MTU 1500, NCCL sits around 3 GB/s.
- The realistic, verified ceiling for two Sparks over RoCE is the **high-teens
  GB/s** range: allgather around 18.8 GB/s, allreduce around 21.5 GB/s. Never
  read the NIC's rated speed as the achieved collective number.

## Prerequisites

- Two DGX Spark systems (Founders Edition; OEM variants differ in places and
  are noted inline).
- One QSFP direct-connect cable.
- Wired USB keyboard and a display (required for first boot and any recovery).
- The same operator username on both nodes.

## Conventions used throughout

- Node names: `spark1`, `spark2`.
- Interface names such as `enp1s0f0np0` are examples; always discover yours
  with `ibdev2netdev`.
- Direct-connect subnets: `192.168.100.0/24` and `192.168.101.0/24`, one per
  NIC so NCCL can stripe across both. This follows NVIDIA's `connect-two-sparks`
  playbook addressing; host numbering is `.10` for `spark1` and `.11` for
  `spark2`.

## Upstream sources

This guide is an independent, verified walkthrough. Where it matters, it cites
and cross-checks the primary sources:

- NVIDIA DGX Spark docs: https://docs.nvidia.com/dgx/dgx-spark/
- NVIDIA DGX Spark playbooks (including the `connect-two-sparks` playbook and
  its `discover-sparks` script): https://github.com/NVIDIA/dgx-spark-playbooks
- NCCL and nccl-tests: https://github.com/NVIDIA/nccl and
  https://github.com/NVIDIA/nccl-tests
- Ray: https://docs.ray.io
- NVIDIA Container Toolkit: https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/
- vLLM: https://docs.vllm.ai

Version numbers in the sections are pinned "as of 2026-09-03" and should be
re-confirmed against the sources above before a fresh build; Ray and vLLM move
especially fast.
