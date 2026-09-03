# 03: NCCL Build and Verification

This section builds NCCL and the nccl-tests suite for Blackwell, sets the
multi-node environment variables, and verifies allgather and allreduce at the
hardware ceiling: allgather busbw around 18.8 GB/s, allreduce busbw around
21.5 GB/s.

What you end up with: both nodes can run a 16 GiB `all_gather_perf` and
`all_reduce_perf` across the QSFP RoCE link, the NCCL debug output proves the
data path is RoCE (not TCP), and the reported busbw sits in the high-teens
GB/s range that is the realistic ceiling for this hardware. The high-teens
number is the target, not 200 GB/s and not 25 GB/s: those are the NIC's rated
speed and per-port theoretical ceiling, not what NCCL achieves on this link.

> Prerequisite: `02-networking.md` is done on both nodes (RoCE interfaces up at
> MTU 9000, static IPs, `ib_write_bw` verified, passwordless SSH over the
> management plane). Run the same build on both nodes. The commands below run
> as your regular user with `sudo` where shown.

---

## 0. Reference versions (verified)

Verified against NVIDIA's `dgx-spark-playbooks` NCCL README and the
`nccl-tests` README as of 2026-09-03.

| Component | Version |
| --- | --- |
| NCCL | v2.30.7-1 (pinned by the playbook; latest GitHub release at writing is v2.31.2-1) |
| nccl-tests | git master (the playbook does not pin a tag) |
| CUDA Toolkit | 13.0.2 (from section 01, DGX OS 7.5.0) |
| OpenMPI | the `libopenmpi-dev` package (Ubuntu 24.04) |

The playbook pins `v2.30.7-1` and calls it "the latest version" as of its
last update (2025-12-15). A newer release now exists. If the nccl-tests build
fails against the pin, try the latest release instead: some users have
reported that switching to a newer NCCL clears a nccl-tests build failure.
Start with the pin; it is the version the playbook validates.

---

## 1. Prerequisites

On both nodes, confirm the CUDA toolkit and the compiler are present:

```bash
nvcc --version
```

Expected output includes the CUDA version (13.0.x on DGX OS 7.5.0, see
section 01). If `nvcc` is not found, install the CUDA toolkit first; DGX OS
normally ships it at `/usr/local/cuda`.

Confirm the build tools are present (they ship with DGX OS):

```bash
gcc --version
make --version
```

If either is missing, install `build-essential`.

NCCL's IB transport needs the RDMA verbs headers at build time. The RDMA
userspace is already present (section 02 used `ibstat` and `ib_write_bw` from
it), but the development headers may not be. If the NCCL build below reports
missing infiniband headers, install them:

```bash
sudo apt-get install -y libibverbs-dev librdmacm-dev
```

Install OpenMPI. This provides both `mpirun` and the MPI headers and
libraries that nccl-tests links against:

```bash
sudo apt-get update && sudo apt-get install -y libopenmpi-dev
```

Confirm `mpirun` is on the path:

```bash
mpirun --version
```

---

## 2. Build NCCL

Clone the pinned release and build with Blackwell support. The
`NVCC_GENCODE` flag targets the GB10 GPU (SM 12.1):

```bash
git clone -b v2.30.7-1 https://github.com/NVIDIA/nccl.git ~/nccl
cd ~/nccl
make -j src.build NVCC_GENCODE="-gencode=arch=compute_121,code=sm_121"
```

This produces the shared library `~/nccl/build/lib/libnccl.so` and the header
`~/nccl/build/include/nccl.h`. The NCCL library itself does not need MPI; the
`MPI=1` flag belongs to the test suite in the next step.

Then export the build and link paths. These are used by the nccl-tests build
and, later, at run time:

```bash
export CUDA_HOME="/usr/local/cuda"
export MPI_HOME="/usr/lib/aarch64-linux-gnu/openmpi"
export NCCL_HOME="$HOME/nccl/build"
export LD_LIBRARY_PATH="$NCCL_HOME/lib:$CUDA_HOME/lib64:$MPI_HOME/lib:$LD_LIBRARY_PATH"
```

Verify the library is there:

```bash
ls -l ~/nccl/build/lib/libnccl.so
```

---

## 3. Build nccl-tests

Clone the test suite and build it with MPI support. The `MPI=1` flag compiles
the test binaries against OpenMPI, which is what lets `mpirun` launch and
coordinate them across the two nodes:

```bash
git clone https://github.com/NVIDIA/nccl-tests.git ~/nccl-tests
cd ~/nccl-tests
make MPI=1
```

The build reads `CUDA_HOME`, `NCCL_HOME`, and `MPI_HOME` from the environment
you exported in section 2, so do not skip that export step. Confirm the
binaries exist:

```bash
ls -l ~/nccl-tests/build/all_gather_perf ~/nccl-tests/build/all_reduce_perf
```

---

## 4. Environment variables

The runtime block below sets two planes explicitly: the data plane (which RoCE
HCAs carry the RDMA traffic) and the control plane (which interface `mpirun`
launches over and NCCL bootstraps on). Set these on the launcher node before
running `mpirun`.

```bash
# Data plane: the RoCE HCAs that carry RDMA traffic. Derive these from the
# LEFT column of ibdev2netdev, comma-separated with no spaces.
export NCCL_IB_HCA="rocep1s0f0,roceP2p1s0f0"

# Control plane: the management interface mpirun SSHs over and NCCL
# bootstraps on. Discover yours with `ip -o link show`. In this guide's
# Tailscale setup that is tailscale0; in NVIDIA's wired-Ethernet playbook it
# is enP7s7.
export NCCL_SOCKET_IFNAME="tailscale0"
export OMPI_MCA_btl_tcp_if_include="tailscale0"
export UCX_NET_DEVICES="tailscale0"

# Paths, used at run time so the test binaries find libnccl.so.
export CUDA_HOME="/usr/local/cuda"
export MPI_HOME="/usr/lib/aarch64-linux-gnu/openmpi"
export NCCL_HOME="$HOME/nccl/build"
export LD_LIBRARY_PATH="$NCCL_HOME/lib:$CUDA_HOME/lib64:$MPI_HOME/lib:$LD_LIBRARY_PATH"

# Verbose enough to see the NET/ transport line in the verify step.
export NCCL_DEBUG=INFO
```

Each variable, explained:

- **`NCCL_IB_HCA`** is the one that matters most for getting RoCE. It lists
  the RoCE device names, taken from the **left** column of `ibdev2netdev`
  output:

  ```text
  rocep1s0f0  port 1 ==> enp1s0f0np0  (Up)
  roceP2p1s0f0 port 1 ==> enP2p1s0f0np0 (Up)
  ```

  Copy the left-hand names exactly, comma-separated, no spaces. Do not use the
  netdev names on the right (`enp1s0f0np0`, `enP2p1s0f0np0`) here. Note the
  capitalization differs between the two devices (`rocep1s0f0` lowercase `p1`
  vs `roceP2p1s0f0` uppercase `P2`), so copy yours, do not retype. Setting this
  forces NCCL's data path onto RoCE; without it NCCL can fall back to TCP
  sockets, which is far slower on this link.

- **`NCCL_SOCKET_IFNAME`** selects the interface NCCL uses for its initial
  socket-based control connection. Point it at the management interface, the
  one whose addresses you SSH to, so the handshake is reachable.

- **`OMPI_MCA_btl_tcp_if_include`** tells OpenMPI which interface its TCP
  transport (the process launch/bootstrap) uses. This must be the management
  interface, the one `mpirun` reaches over.

- **`UCX_NET_DEVICES`** restricts UCX to one device, if OpenMPI was built with
  UCX support. NVIDIA's playbook points it at the management interface; it is
  a no-op on builds that do not use UCX.

- **`CUDA_HOME`**, **`MPI_HOME`**, **`NCCL_HOME`**, **`LD_LIBRARY_PATH`** are
  the build/link paths from section 2, re-exported so the test binaries find
  `libnccl.so.2` at run time.

Interface names vary by node and by which physical ports you used, so confirm
yours rather than copying the examples. The management interface is the one
whose IP or hostname you use to SSH between nodes; list candidates with
`ip -o link show`. The RoCE device names come only from `ibdev2netdev`.

---

## 5. Launch

Create a hostfile listing the two nodes (one slot each, one GPU per node):

```text
spark1 slots=1
spark2 slots=1
```

If the hostnames do not resolve over the management plane, substitute the
management IPs (for example the Tailscale addresses from section 01).

The `LD_LIBRARY_PATH` gotcha, fixed. The naive form
`-x LD_LIBRARY_PATH=$LD_LIBRARY_PATH` expands locally, so if the variable is
unset it forwards an empty value to the remote rank and the test dies with
`libnccl.so.2: cannot open shared object file`. The fix is two parts: export
`LD_LIBRARY_PATH` first (done in section 4), then pass `-x LD_LIBRARY_PATH`
with **no** `=$...` suffix, which exports the current value:

```bash
mpirun --hostfile hostfile -np 2 \
  --mca plm_rsh_agent "ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no" \
  -x LD_LIBRARY_PATH \
  $HOME/nccl-tests/build/all_gather_perf -b 16G -e 16G -f 2
```

Notes:

- `--hostfile hostfile` launches one rank per line (two ranks total). The
  equivalent inline form is `-H spark1:1,spark2:1`.
- `--mca plm_rsh_agent "ssh ..."` makes `mpirun` launch over SSH without
  host-key prompts; it is optional once the nodes are in `known_hosts`, but
  keeps first runs from hanging on a prompt.
- `-x LD_LIBRARY_PATH` (no value) exports the launcher's current value to the
  remote ranks. This is the corrected form; do not append `=$LD_LIBRARY_PATH`.
- `-b 16G -e 16G -f 2` runs a single 16 GiB message size per rank. With
  `-b == -e` the `-f 2` stepping factor is ignored. 16 GiB is the size that
  produced the verified numbers; it fits comfortably in the 128 GB unified
  memory.

---

## 6. Verify

Run the two collectives at the 16 GiB buffer. All-gather first:

```bash
mpirun --hostfile hostfile -np 2 \
  --mca plm_rsh_agent "ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no" \
  -x LD_LIBRARY_PATH \
  $HOME/nccl-tests/build/all_gather_perf -b 16G -e 16G -f 2
```

Then all-reduce:

```bash
mpirun --hostfile hostfile -np 2 \
  --mca plm_rsh_agent "ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no" \
  -x LD_LIBRARY_PATH \
  $HOME/nccl-tests/build/all_reduce_perf -b 16G -e 16G -f 2
```

### Reading the output

The perf table header is stable across nccl-tests versions:

```text
#                                                              out-of-place                       in-place
#       size         count      type   redop    root     time   algbw   busbw      #wrong     time   algbw   busbw      #wrong
#        (B)    (elements)                               (us)  (GB/s)  (GB/s)                  (us)  (GB/s)  (GB/s)
```

Read the **busbw** column. Expect roughly 18.8 GB/s for all-gather and
roughly 21.5 GB/s for all-reduce. The `#wrong` column reports how many
elements failed the correctness check; it should read 0 on both sides (the
`out-of-place` and `in-place` results). Anything in the high-teens GB/s range
means the fabric is at its ceiling. If you see around 3 GB/s instead, skip to
the diagnostic ladder in section 7; the cause is almost certainly MTU 1500.

### Proving RoCE (not TCP)

`NCCL_DEBUG=INFO` was exported in section 4, so each rank prints its transport
decision. Two lines matter:

1. The transport line. Grep the run for the NET transport and confirm it is
   the IB/RoCE path, not sockets:

   ```bash
   mpirun ... all_gather_perf -b 16G -e 16G -f 2 2>&1 | grep -i "NET/"
   ```

   You want a line that names the RoCE devices and says `NET/IB` (or lists the
   `rocep1s0f0` / `roceP2p1s0f0` devices). If it says `NET/Socket` or
   `NET/TCP` instead, NCCL fell back to TCP and your data path is wrong. The
   exact wording of this line varies by NCCL version, so match on the `NET/IB`
   token and the device names rather than a specific sentence.

2. The system summary line that reports NCCL's computed ceiling, of the form
   `=== System : maxBw <X> totalBw <Y> ===`. This appears once per node in the
   INFO output and reflects the bandwidth NCCL derived from the topology. It is
   a sanity check, not a target: a clean high-teens busbw with a `NET/IB` line
   is what proves the cluster is working, regardless of what `maxBw` prints.

A passing run looks like: busbw in the high teens, `#wrong` of 0, and a
`NET/IB` line naming the RoCE devices.

---

## 7. Diagnostic ladder (cheap to expensive)

If the numbers come out around 3 GB/s instead of the high teens, work through
these in order. The 3 GB/s symptom (allgather algbw around 6.0, busbw around
3.0) while the link otherwise looks fine is the classic MTU 1500 case, so
check that first.

1. **MTU 9000.** On both nodes, every direct-connect interface must read
   `mtu 9000`:

   ```bash
   ip link show enp1s0f0np0
   ip link show enP2p1s0f0np0
   ```

   If any reads `mtu 1500`, raise it on both nodes (matching interfaces), then
   re-run:

   ```bash
   sudo ip link set dev enp1s0f0np0 mtu 9000
   sudo ip link set dev enP2p1s0f0np0 mtu 9000
   ```

   MTU 1500 is the most common cause of a 3 GB/s cap. Persist the fix in
   netplan (section 02), and re-check after any reboot or firmware update,
   since firmware updates have been observed to reset MTU silently.

2. **The cosmetic PCIe misreport.** Query the link state:

   ```bash
   nvidia-smi --query-gpu=pcie.link.gen.current,pcie.link.width.current,pcie.link.gen.max,pcie.link.width.max --format=csv
   ```

   Expect `1, 1, 1, 16` (Gen1 x1, current and max). This is a known cosmetic
   misreport and does not throttle NCCL: verified runs showing this exact
   output still reached 18.8 GB/s. Note it and move on. Do not reach for
   `NCCL_TOPO_FILE` overrides or a firmware change to fix it.

3. **Raw RDMA throughput.** Confirm the link itself is fast with
   `ib_write_bw` (see section 02 for the full command). Expect 100+ Gbps from
   64 KB upward, peaking near 196 Gbps at the largest sizes. A `syndrom 0x88`
   crash at the 128 KB message size is a known test-harness artifact
   (single-QP reliable connection), not a fabric fault; NCCL's data path is
   unaffected.

4. **Re-run the real buffer.** After the checks above, run the 16 GiB
   `all_gather_perf` again. A clean high-teens busbw with a `NET/IB` line
   means the cluster is at its hardware ceiling.

---

## 8. Expected result

- `nvcc --version` reports CUDA 13.0.x and `mpirun --version` reports OpenMPI
  on both nodes.
- `~/nccl/build/lib/libnccl.so` and the `all_gather_perf` / `all_reduce_perf`
  binaries exist on both nodes.
- `all_gather_perf -b 16G -e 16G -f 2` reports busbw around 18.8 GB/s with
  `#wrong` of 0.
- `all_reduce_perf -b 16G -e 16G -f 2` reports busbw around 21.5 GB/s with
  `#wrong` of 0.
- The `NCCL_DEBUG=INFO` output shows a `NET/IB` line naming the RoCE devices,
  not `NET/Socket`.

If busbw is in the high teens and the transport is `NET/IB`, the two-node
fabric is done. The next sections (Ray, vLLM) build on this verified NCCL
path.
