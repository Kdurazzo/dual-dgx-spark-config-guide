# 04: Ray Two-Node Cluster

This section installs Ray on both nodes, brings up a two-node Ray cluster
(head on `spark1`, worker on `spark2`) on the management plane, and verifies a
distributed task runs on both GPUs. Ray is the control-plane backend that
`06-vllm.md` later uses to place a two-GPU tensor-parallel model, so this
section ends with what must be in place for that handoff.

What you end up with: a Ray head on `spark1`, a worker on `spark2`, the
dashboard reachable over Tailscale, `ray status` reporting 2 nodes and 2 GPUs,
and a small task that prints a different hostname and its local GPU from each
node.

> Prerequisite: `01-os-bringup.md` and `02-networking.md` are done on both
> nodes (OS installed, SSH keys and Tailscale reachable, RoCE data plane up at
> MTU 9000). Commands run as your regular user with `sudo` where shown.

---

## 1. Reference version (verified)

Verified against the Ray GitHub releases page and docs.ray.io as of
2026-09-03.

| Component | Version |
| --- | --- |
| Ray | 2.58.0 (latest stable, released 2026-08-23) |
| Supported platforms | Linux x86_64, Linux aarch64, Apple silicon |
| Supported Python | 3.10 through 3.14 (aarch64 wheels included) |

Source: https://github.com/ray-project/ray/releases and
https://docs.ray.io/en/latest/ray-overview/installation.html. Ray 2.58.0 is the
newest stable release as of the writing date; point releases move quickly, so
confirm against those pages rather than treating the number as fixed.

Ray officially supports Linux aarch64, and PyPI ships prebuilt
`manylinux2014_aarch64` wheels for Python 3.10 through 3.14. DGX OS 7.5.0 is
Ubuntu 24.04 LTS based and its system Python is 3.12, which is in that range,
so `pip` fetches a prebuilt wheel and no source build is needed on Grace.

---

## 2. Install (both nodes)

Run the same steps on `spark1` and `spark2`. Use a dedicated virtual
environment so Ray and its dependencies do not pollute the system Python, and
so the same environment exists identically on both nodes (a distributed Ray
task runs in the environment of whichever node executes it).

```bash
# Confirm the interpreter version; expect 3.12.x on DGX OS 7.5.0
python3 --version

# venv/pip are normally present, but install them if the command below fails
sudo apt install -y python3-venv python3-pip

python3 -m venv ~/ray-venv
source ~/ray-venv/bin/activate
pip install -U pip
pip install -U "ray[default]"
```

The `[default]` extra includes the dashboard and the cluster launcher, both of
which this section uses. To check the install and pin the version you have:

```bash
ray --version
```

Expected output on each node:

```
ray, version 2.58.0
```

Notes:

- Installing into a venv is the officially documented method for Linux. There
  is no separate Ray-provided installer for this OS; the installer is `pip`.
- Both nodes need the venv, because `ray start` runs local Ray processes
  (raylet, object store, workers) on every node, not only on the head.
- Keep the venv path identical on both nodes (`~/ray-venv`) so any later
  scripts that assume a path stay consistent across the cluster.

---

## 3. Cluster configuration

### 3.1 Which plane: use Tailscale, not the RoCE subnet

Ray's control traffic (the GCS global control store, the dashboard, and the
gRPC connections between nodes) is ordinary TCP. It does not use RDMA, so it
has nothing to gain from the RoCE data plane. Run Ray on the Tailscale
management plane:

- Tailscale gives every node a stable, routable address that your laptop can
  also reach, which is what the dashboard and `ray.init()` from a client need.
- The RoCE direct-connect subnets (`192.168.100.0/24` and `192.168.101.0/24`) are
  point-to-point between the two ConnectX-7 NICs, with no gateway, no DNS, and
  no route off the box. A Ray cluster bound to those addresses could not be
  reached from anywhere else and would stop working the moment you touched the
  data plane.

So the management plane carries Ray, and the data plane stays reserved for
NCCL and the vLLM tensor-parallel traffic in sections 03 and 06.

Find each node's Tailscale IPv4 address:

```bash
tailscale ip -4
```

Use those addresses below. The examples write them as `<spark1-ts>` and
`<spark2-ts>`; substitute your real values.

### 3.2 Start the head on spark1

```bash
source ~/ray-venv/bin/activate
ray start --head \
  --port=6379 \
  --node-ip-address=<spark1-ts> \
  --dashboard-host 0.0.0.0
```

Flag meanings:

- `--head` makes this node the cluster head (it hosts the GCS and the
  scheduler).
- `--port=6379` sets the GCS port. 6379 is the default; it is set explicitly
  here so the worker's `--address` below is unambiguous.
- `--node-ip-address=<spark1-ts>` pins the address the head advertises. A
  Spark has at least four interfaces (two RoCE, Tailscale, and possibly a LAN
  port), and Ray's auto-detection can pick the wrong one; pinning avoids that.
- `--dashboard-host 0.0.0.0` binds the dashboard to all interfaces so you can
  reach it from your laptop over Tailscale (section 5). By default the
  dashboard listens only on the loopback address.

Expected output includes a line like:

```
View the Ray dashboard at http://127.0.0.1:8265
```

### 3.3 Start the worker on spark2

```bash
source ~/ray-venv/bin/activate
ray start --address=<spark1-ts>:6379 \
  --node-ip-address=<spark2-ts>
```

`--address` points the worker at the head's GCS. `--node-ip-address` pins the
address this worker advertises back to the head, for the same multi-interface
reason as the head. Expected output ends with:

```
This node has been joined to the cluster.
```

The worker needs no `--head` and no `--port`; it registers with the head and
learns the rest from it.

### 3.4 Confirm the cluster is up

On the head:

```bash
ray status
```

You want two nodes and, for now, whatever CPU/memory Ray auto-detected. GPU
detection is covered next, but at this point the cluster should already list
`spark1` and `spark2` as separate nodes.

---

## 4. GPU resource detection

Ray detects GPUs at startup via the NVIDIA driver and reports them as the
`GPU` resource. Each Spark has one Blackwell GPU, so each node contributes
`1.0` to the `GPU` resource and the cluster total is `2.0`.

Check from the head:

```bash
ray status
```

Expected shape (CPU counts will reflect the 10-core GB10 CPU):

```
======== Cluster Resources ========
Resources
---------
Usage: 0.0/2.0 GPUs, 0/20 CPU, ...

Nodes
-----
2 nodes

Node spark1: 1 GPU, 10 CPU
Node spark2: 1 GPU, 10 CPU
```

If a node reports `0 GPU`, check that `nvidia-smi` works on that node and that
Ray was started inside the venv with the NVIDIA driver visible. Ray reads GPU
count through the driver at startup; if you started Ray before the driver
loaded, run `ray stop` and start it again.

In code, the current API for reading the GPUs a task or actor was assigned is
the runtime context, not the older `ray.get_gpu_ids()` (which is deprecated):

```python
import ray

ray.init(address="auto")          # connect to the running cluster, do not start a new one

print(ray.cluster_resources())    # dict with "GPU": 2.0, "CPU": 20.0, ...
```

Inside a task or actor:

```python
gpu_ids = ray.get_runtime_context().get_accelerator_ids("GPU")
# returns a list of strings, e.g. ["0"] on a single-GPU node
```

The deprecated equivalent is `ray.get_gpu_ids()`; prefer
`get_accelerator_ids("GPU")`, which returns strings and is the documented
replacement (Ray issue #44820 records the deprecation).

---

## 5. Dashboard

The dashboard runs on the head at port 8265. With `--dashboard-host 0.0.0.0`
from section 3.2, it is reachable from any machine on the Tailnet:

```
http://<spark1-ts>:8265
```

Open it from your laptop. You should see the cluster with two nodes and
`2.0 GPUs` in the resources summary. If you do not want the dashboard, restart
the head with `--include-dashboard=false`; the cluster still works, you just
lose the web UI.

---

## 6. Verify: a task on both GPUs

Save the following on the head (any node on the Tailnet works, since it
connects over Tailscale) and run it inside the venv:

```python
# verify_ray.py
import socket
import ray

ray.init(address="auto")

@ray.remote(num_gpus=1)
def whoami():
    ctx = ray.get_runtime_context()
    return {
        "hostname": socket.gethostname(),
        "gpus": ctx.get_accelerator_ids("GPU"),
    }

# Two tasks, each requesting 1 GPU. Each node has exactly one GPU, so Ray is
# forced to place one task on spark1 and one on spark2.
results = ray.get([whoami.remote() for _ in range(2)])
for r in sorted(results, key=lambda d: d["hostname"]):
    print(r)
```

Run it:

```bash
source ~/ray-venv/bin/activate
python verify_ray.py
```

Expected output, two entries with different hostnames and one GPU each:

```
{'hostname': 'spark1', 'gpus': ['0']}
{'hostname': 'spark2', 'gpus': ['0']}
```

Then confirm the cluster-level view one more time:

```bash
ray status
```

Acceptance: `ray status` shows 2 nodes and 2 total GPUs, and the task above
returns one result from `spark1` and one from `spark2`, each listing GPU `0`.

If the two results came back with the same hostname, the worker did not join
correctly; re-check section 3.3 and that `ray status` shows two nodes before
re-running.

---

## 7. Notes for the vLLM handoff

Section 06 uses Ray to place vLLM's two tensor-parallel ranks, one per node.
Before that section, the following must already be in place (all done above):

- Ray head running on `spark1` and reachable at a stable address over
  Tailscale (`<spark1-ts>:6379`), which is the address vLLM will be pointed at.
- Worker `spark2` joined and both nodes listed by `ray status`.
- Both GPUs visible, so `ray.cluster_resources()["GPU"]` reports `2.0` and
  each node contributes one GPU.
- Ray and vLLM installed in a compatible Python environment on both nodes
  (vLLM imports Ray, so the two must share an environment or use matching
  versions). The exact vLLM install and the address/flag it takes are covered
  in `06-vllm.md`, not here.

One distinction to keep straight for the handoff: Ray only schedules and
places the vLLM ranks. The actual tensor-parallel traffic between the two
GPUs (weight transfer and the allreduce during inference) flows over the RoCE
data plane via NCCL, not through Ray. That is why section 02's MTU 9000
fabric matters even though Ray itself runs on Tailscale.

---

## Done criteria

- `ray --version` returns 2.58.0 (or your pinned later release) on both nodes.
- `ray status` shows 2 nodes, with `spark1` as the head and `spark2` joined.
- `ray.cluster_resources()` reports `"GPU": 2.0`.
- The dashboard is reachable at `http://<spark1-ts>:8265` from your laptop.
- `verify_ray.py` returns one result from `spark1` and one from `spark2`, each
  listing GPU `0`.

To tear the cluster down later (in this order), run `ray stop` on the worker
first, then on the head.
