# 07: Acceptance and Troubleshooting

This section is the last gate. It compresses sections 01 through 06 into one
end-to-end checklist you can run top to bottom to confirm a healthy two-node
cluster, then a troubleshooting appendix for the failure modes that bite most
often, and finally a quick-reference block with each environment variable and
verified number in one place for copy-paste.

What you end up with: a single pass/fail procedure from first boot to a live
two-GPU inference response, plus a fix you can apply for each known failure
without reading the rest of the guide.

> Prerequisite: sections 01 through 06 are done. The checklist is the
> verification, not the build; if a gate fails, the troubleshooting appendix
> and the referenced section tell you what to fix.

---

## 1. End-to-end acceptance checklist

Run these gates in order. Each names the command, the expected output, and the
section that built that piece. A healthy cluster passes all seven.

### Gate 1: Hostnames and passwordless SSH (section 01)

(Section 01 also documents the `discover-sparks` script as a faster alternative
to manual key setup; the manual path is the primary one this checklist
verifies.)

On each node, confirm the hostname:

```bash
hostnamectl
```

Expected: `Static hostname: spark1` on `spark1`, `Static hostname: spark2` on
`spark2`.

Confirm passwordless SSH works in both directions. From `spark1`:

```bash
ssh spark2 hostname
```

Expected: `spark2` with no password prompt. From `spark2`:

```bash
ssh spark1 hostname
```

Expected: `spark1` with no password prompt.

### Gate 2: RoCE interfaces up, Rate 200, MTU 9000 persisted (section 02)

Map the RoCE devices to Linux interfaces:

```bash
ibdev2netdev
```

Expected: two interfaces report `(Up)` per node (one cable lights up both
port-1 interfaces through the on-die eSwitch).

Confirm link state and rate on each RoCE device:

```bash
ibstat rocep1s0f0
ibstat roceP2p1s0f0
```

Expected: `LinkUp` and `Rate: 200` on each. The device names are examples;
copy yours from `ibdev2netdev`.

Confirm MTU 9000 is set and persists:

```bash
ip link show enp1s0f0np0
ip link show enP2p1s0f0np0
```

Expected: `mtu 9000` and `state UP` on each direct-connect interface. Re-run
this after a reboot to confirm the netplan `mtu: 9000` persisted rather than
reverting to 1500.

### Gate 3: Raw RDMA throughput (section 02)

On `spark2` (server):

```bash
ib_write_bw -d rocep1s0f0 -a --report_gbits
```

On `spark1` (client):

```bash
ib_write_bw -d rocep1s0f0 192.168.100.11 -a --report_gbits
```

Expected: roughly 100+ Gbps from 64 KB upward, peaking near 196 Gbps at the
largest sizes. Repeat against the second NIC pair (`roceP2p1s0f0`, peer
`192.168.101.11`) to confirm both NICs carry traffic.

### Gate 4: NCCL allgather and allreduce (section 03)

Run the 16 GiB all-gather from the launcher node (env vars already exported):

```bash
mpirun --hostfile hostfile -np 2 \
  --mca plm_rsh_agent "ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no" \
  -x LD_LIBRARY_PATH \
  $HOME/nccl-tests/build/all_gather_perf -b 16G -e 16G -f 2
```

Expected: `busbw` around 18.8 GB/s, `#wrong` of 0, and a `NET/IB` line naming
the RoCE devices (not `NET/Socket`).

Then all-reduce:

```bash
mpirun --hostfile hostfile -np 2 \
  --mca plm_rsh_agent "ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no" \
  -x LD_LIBRARY_PATH \
  $HOME/nccl-tests/build/all_reduce_perf -b 16G -e 16G -f 2
```

Expected: `busbw` around 21.5 GB/s, `#wrong` of 0.

### Gate 5: Ray sees 2 nodes and 2 GPUs (section 04)

On the head (`spark1`):

```bash
source ~/ray-venv/bin/activate
ray status
```

Expected: `2 nodes` and a cluster total of `2.0 GPUs`, with `spark1` and
`spark2` each contributing one GPU.

### Gate 6: Docker sees the GPU (section 05)

On each node:

```bash
sudo docker run --rm --runtime=nvidia --gpus all ubuntu nvidia-smi
```

Expected: the NVIDIA GB10 GPU line, with the driver version matching the host
and `CUDA Version: 13.0`. Run this on both `spark1` and `spark2`; a
single-node pass does not prove the pair.

### Gate 7: vLLM inference with both GPUs active (section 06)

From any machine on the Tailnet (your laptop works), confirm the model is
served:

```bash
curl http://<spark1-ts>:8000/v1/models
```

Expected: a JSON list whose `data` array names the served model (for example
`Qwen/Qwen2.5-7B-Instruct`).

Send one inference request:

```bash
curl http://<spark1-ts>:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen2.5-7B-Instruct",
    "messages": [
      {"role": "user", "content": "Reply with exactly: two nodes"}
    ],
    "max_tokens": 16
  }'
```

Expected: a `choices` array with an `assistant` message and generated text.

Confirm both GPUs participated while the server runs:

```bash
ray status
```

Expected: `Usage: 2.0/2.0 GPUs`.

```bash
nvidia-smi
```

Expected: on both `spark1` and `spark2`, the GB10 shows a large memory
reservation and nonzero compute activity during a request. Memory in use on
both nodes at once is the concrete proof the model is split, not served
twice.

---

## 2. Troubleshooting appendix

The six failure modes that bite most often, with the cause and the fix. Each
fix stands alone; you do not need to read the rest of the guide to apply it.

| Symptom | Cause | Fix |
| --- | --- | --- |
| NCCL allgather/allreduce sits at about 3 GB/s (algbw about 6.0, busbw about 3.0) while the link looks fine | MTU 1500 on a direct-connect interface | Set MTU 9000 on both nodes, matching interfaces, persist it, re-run |
| `nvidia-smi` reports PCIe link as Gen1 x1 (query shows `1, 1, 1, 16`) | Cosmetic misreport on the GB10 | Ignore it; it does not throttle NCCL. Do not reach for `NCCL_TOPO_FILE` overrides or firmware |
| `ib_write_bw` crashes at the 128 KB message size with `syndrom 0x88` | Test-harness artifact (single-QP reliable connection) | Ignore it; NCCL's data path is unaffected. Verify with the NCCL 16 GiB test instead |
| MTU reads 1500 again after a firmware update or reboot | Firmware updates have been observed to reset MTU silently | Re-check `ip link show` after each firmware update and reboot; re-apply MTU 9000 if it reverted |
| `mpirun` dies with `libnccl.so.2: cannot open shared object file` | The `-x LD_LIBRARY_PATH=$LD_LIBRARY_PATH` form expands locally, forwarding empty if unset | Export `LD_LIBRARY_PATH` first, then pass `-x LD_LIBRARY_PATH` with no `=$...` suffix |
| A node needs a full reinstall | There is no CLI factory reset and no BMC/IPMI on the Spark | Physical recovery only: USB keyboard, display, and a 16 GB+ USB drive built from NVIDIA's recovery media (FE tar.gz), then `[START RECOVERY]` |

The two fixes that need full commands:

MTU 9000, applied immediately (ephemeral) and persisted. On both nodes, on
matching interfaces:

```bash
sudo ip link set dev enp1s0f0np0 mtu 9000
sudo ip link set dev enP2p1s0f0np0 mtu 9000
```

The `ip link set` form is lost on reboot. The durable fix is the `mtu: 9000`
line in the netplan file from section 02; after re-applying, re-run
`ip link show` to confirm it stuck.

The `LD_LIBRARY_PATH` gotcha, fixed. First export the path (section 03):

```bash
export LD_LIBRARY_PATH="$HOME/nccl/build/lib:/usr/local/cuda/lib64:/usr/lib/aarch64-linux-gnu/openmpi/lib:$LD_LIBRARY_PATH"
```

Then pass `-x LD_LIBRARY_PATH` to `mpirun` with no `=$...` suffix, so it
forwards the current value to the remote ranks rather than an empty string.

---

## 3. Quick reference

Each environment variable in one block. Set these on the launcher node
before running NCCL tests, and on `spark1` before `vllm serve` (the Ray
variables apply to the vLLM launch).

```bash
# Data plane: the RoCE HCAs carrying RDMA traffic. Copy these from the LEFT
# column of ibdev2netdev, comma-separated, no spaces.
export NCCL_IB_HCA="rocep1s0f0,roceP2p1s0f0"

# Control plane: the management interface the NCCL bootstrap and mpirun
# launch use. In this guide's Tailscale setup that is tailscale0.
export NCCL_SOCKET_IFNAME="tailscale0"
export OMPI_MCA_btl_tcp_if_include="tailscale0"
export UCX_NET_DEVICES="tailscale0"

# Ray: connect to the running cluster instead of starting one.
export RAY_ADDRESS="<spark1-ts>:6379"

# vLLM TP and gloo bootstrap interfaces (honored by newer vLLM releases).
export TP_SOCKET_IFNAME="tailscale0"
export GLOO_SOCKET_IFNAME="tailscale0"

# Paths, used at run time so binaries find libnccl.so.
export CUDA_HOME="/usr/local/cuda"
export MPI_HOME="/usr/lib/aarch64-linux-gnu/openmpi"
export NCCL_HOME="$HOME/nccl/build"
export LD_LIBRARY_PATH="$NCCL_HOME/lib:$CUDA_HOME/lib64:$MPI_HOME/lib:$LD_LIBRARY_PATH"

# Verbose enough to see the NET/ transport line.
export NCCL_DEBUG=INFO
```

The verified numbers, in one place:

| Metric | Value |
| --- | --- |
| ConnectX-7 rated speed | 200 Gbps per port (about 25 GB/s theoretical) |
| PCIe link per NIC | PCIe 5.0 x4 (about 12 GB/s effective each) |
| LPDDR5x unified memory bandwidth | 273 GB/s (memory, not network) |
| Raw RDMA `ib_write_bw` | about 100+ Gbps from 64 KB, peaking near 196 Gbps |
| NCCL allgather busbw | about 18.8 GB/s |
| NCCL allreduce busbw | about 21.5 GB/s |
| NCCL at MTU 1500 (fault state) | about 3.0 GB/s |

The realistic ceiling for a two-Spark cluster is the high-teens GB/s range.
Never read 200 Gbps or 25 GB/s as the achieved NCCL number; those are the
NIC's rated speed and per-port theoretical ceiling, not what collectives
deliver over this link.

---

## 4. Done criteria

The cluster is accepted when you can answer yes to all of these, in order:

- Passwordless SSH works both ways, and each node reports its intended
  hostname.
- Each direct-connect interface shows `mtu 9000` and `state UP`, `ibstat`
  reports `Rate: 200`, and the setting survives a reboot.
- `ib_write_bw` shows roughly 100+ Gbps on both NIC pairs.
- `all_gather_perf -b 16G` reports busbw around 18.8 GB/s and
  `all_reduce_perf -b 16G` around 21.5 GB/s, both with `#wrong` of 0 and a
  `NET/IB` transport line.
- `ray status` shows 2 nodes and 2.0 GPUs.
- `docker run --runtime=nvidia --gpus all ubuntu nvidia-smi` prints the GB10
  GPU on both nodes.
- The vLLM endpoint returns a generated response, `ray status` shows
  `Usage: 2.0/2.0 GPUs`, and `nvidia-smi` shows GPU memory in use on both
  nodes.

If a gate fails, the troubleshooting appendix names the likely cause and the
fix, and the referenced section has the full procedure.
