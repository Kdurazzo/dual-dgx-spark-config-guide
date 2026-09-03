# 06: vLLM Multi-Node Tensor Parallel

This section installs vLLM on both nodes and serves one model split across the
two Blackwell GPUs using tensor parallelism (TP), backed by the Ray cluster
from `04-ray.md`. The two tensor-parallel ranks land one per node, and the
inter-GPU traffic between them flows over the RoCE fabric verified in
`03-nccl.md`.

What you end up with: a single OpenAI-compatible endpoint on `spark1` serving a
model whose weights are sharded half on `spark1` and half on `spark2`, a
successful inference response, and evidence that both GPUs participated.

> Prerequisite: `04-ray.md` is done (Ray head on `spark1`, worker `spark2`
> joined, `ray status` shows 2 nodes and 2 GPUs) and `02-networking.md` is done
> (RoCE fabric up at MTU 9000). vLLM installs into the same venv as Ray, so
> sections 01 through 05 must all be in place first. Run installs on both
> nodes; launch the server from `spark1` only.

---

## 0. Reference versions (verified)

Verified against the vLLM GitHub releases page, docs.vllm.ai, and the official
vLLM "vLLM on the DGX Spark" blog post as of 2026-09-03.

| Component | Version / fact |
| --- | --- |
| vLLM | 0.28.0 (latest, released 2026-08-26) |
| Install path | `pip install vllm` (PyPI, CUDA 13.0), prebuilt wheel |
| arm64 support | Yes: CUDA 12.9 and 13.0 wheels published for arm64 |
| GB10 target | `sm_121` (Blackwell consumer); vLLM ships and validates for it |
| CLI flag | `--tensor-parallel-size` (hyphenated) in `vllm serve` |

Sources:

- Releases: https://github.com/vllm-project/vllm/releases
- Parallelism docs: https://docs.vllm.ai/en/stable/serving/parallelism_scaling/
- `vllm serve` CLI reference: https://docs.vllm.ai/en/stable/cli/serve/
- DGX Spark blog: https://vllm.ai/blog/2026-06-01-vllm-dgx-spark

Point releases move quickly; confirm the number and flags against those pages
rather than treating the table as fixed.

One flag note to get right up front: the CLI flag in `vllm serve` is
`--tensor-parallel-size` (hyphenated). The underscore form `tensor_parallel_size`
is the Python API keyword argument in `vllm.LLM(...)`, not a renamed CLI flag.
The current stable docs still show the hyphenated CLI form in every example.
If in doubt, run `vllm serve --help` on the installed version and read the
spelling there.

---

## 1. What tensor parallelism means here

Tensor parallelism shards each layer's weight matrices across GPUs. Each GPU
holds a slice of every layer, computes its slice of the activations, and the
ranks exchange partial results with an allreduce on every layer. In this
two-node setup there is one GPU per node, so each Spark holds roughly half the
model, and the allreduce between the two slices runs over the RoCE direct
link. That cross-node allreduce is what makes this a true two-node run rather
than two independent single-node servers.

Ray only places and launches the two ranks. The TP traffic itself is NCCL over
RoCE, exactly the data path section 03 verified at high-teens GB/s. This is
why MTU 9000 still matters even though the Ray control plane runs on Tailscale.

---

## 2. Install (both nodes)

vLLM installs into the same virtual environment as Ray. vLLM imports Ray to use
the distributed executor, so the two must live in one environment with
compatible versions. Use the venv already created in `04-ray.md`, on both
nodes:

```bash
source ~/ray-venv/bin/activate
pip install -U pip
pip install -U "vllm==0.28.0"
```

Notes:

- The PyPI wheel for vLLM 0.28.0 is multi-platform and publishes arm64, so
  `pip` fetches a prebuilt wheel on Grace with no source build. This matches
  the Ray install in section 04, which also used a prebuilt aarch64 wheel.
- Both nodes need the install, because the Ray executor spawns a vLLM worker
  process on whichever node each rank lands on. Keep the venv path identical
  (`~/ray-venv`), as in section 04.
- Version drift gotcha: `pip install vllm` pulls its own Ray pin as a
  dependency, which can change the Ray version in the venv. The Ray cluster
  from section 04 is already running, so confirm the installed Ray still
  matches what the cluster started under before launching:

  ```bash
  ray --version
  ```

  If it changed, stop and restart the cluster from section 04 under the new
  version before running vLLM, so the vLLM driver and the running Ray cluster
  agree.

- The container alternative is vLLM's official OpenAI image
  (`vllm/vllm-openai:v0.28.0`), which the DGX Spark blog recommends running on
  the CUDA 13 track for `sm_121`. That path needs the RoCE interfaces exposed
  to the container (`--network host` plus RDMA devices), per section 05's
  optional networking note. The venv path is the simpler match for section 04's
  Ray install, so it is the primary recommendation here.

---

## 3. Model choice

Pick a small, permissively licensed dense model to prove the two-node path
cheaply and reliably. A 7B to 8B model in BF16 is roughly 14 GB of weights,
which is far inside the 128 GB unified memory pool on a single Spark, so the
demonstration exercises the TP plumbing rather than stressing memory.

Recommended example: `Qwen/Qwen2.5-7B-Instruct`. It is Apache 2.0 licensed,
public on Hugging Face, needs no access token, and is a standard dense
decoder-only model that vLLM supports for tensor parallelism.

Download and auth:

- For `Qwen/Qwen2.5-7B-Instruct` no token is needed. vLLM downloads the weights
  into the Hugging Face cache on first launch, so both nodes need network
  access to Hugging Face.
- A gated model (for example a Llama or Gemma checkpoint) additionally
  requires accepting the terms and authenticating on each node, since a rank
  may fetch weights locally. On each node run `huggingface-cli login` (or set
  the `HF_TOKEN` environment variable) before launching.

Because the model is a Hugging Face repo ID, vLLM fetches it once and the ranks
share it. If your nodes cannot reach Hugging Face from both hosts, download the
checkpoint once and pass the same local path on both nodes with
`--model /path/to/model`.

---

## 4. Environment for the launch

Two planes are involved, exactly as in section 03: the data plane (RoCE, for
the TP allreduce) and the control plane (Tailscale, for the NCCL bootstrap and
the Ray handshake). Export these in the shell before running `vllm serve`, on
`spark1`:

```bash
# Ray: connect to the running cluster from section 04 instead of starting one.
export RAY_ADDRESS="<spark1-ts>:6379"

# Data plane: the RoCE HCAs carrying the TP allreduce. Copy these from the
# LEFT column of ibdev2netdev, comma-separated, no spaces (section 02/03).
export NCCL_IB_HCA="rocep1s0f0,roceP2p1s0f0"

# Control plane: the management interface the NCCL bootstrap runs over. In
# this guide's Tailscale setup that is tailscale0 (section 03). Newer vLLM
# releases also honor TP_SOCKET_IFNAME and GLOO_SOCKET_IFNAME for the TP and
# gloo bootstrap interfaces; point them at the same management interface.
export NCCL_SOCKET_IFNAME="tailscale0"
export TP_SOCKET_IFNAME="tailscale0"
export GLOO_SOCKET_IFNAME="tailscale0"
```

Interface names vary by node; confirm yours with `ibdev2netdev` and `ip link`
rather than copying the examples. `NCCL_IB_HCA` is the one that matters for
throughput: without it, NCCL can fall back to TCP sockets over the management
plane, which is far slower on this link (section 03 covers the same fallback).

---

## 5. Launch with tensor parallelism

Run on `spark1`, inside the venv, after the exports in section 4:

```bash
source ~/ray-venv/bin/activate

vllm serve Qwen/Qwen2.5-7B-Instruct \
  --tensor-parallel-size 2 \
  --distributed-executor-backend ray \
  --dtype bfloat16 \
  --gpu-memory-utilization 0.85 \
  --max-model-len 8192 \
  --host 0.0.0.0 \
  --port 8000
```

Flag meanings:

- `--tensor-parallel-size 2` requests two TP ranks, one model split across two
  GPUs.
- `--distributed-executor-backend ray` tells vLLM to place those ranks through
  the Ray cluster instead of local multiprocessing. With `RAY_ADDRESS` set, it
  joins the section 04 cluster; without it, vLLM would start its own
  single-node Ray and the second rank could not reach `spark2`.
- `--gpu-memory-utilization 0.85` leaves headroom in the unified memory pool.
  On the Spark the CPU, OS, and container runtime share the same 128 GB as the
  GPU, so do not set this near 1.0 (the DGX Spark blog makes this point
  explicitly).
- `--max-model-len 8192` keeps the KV cache small for a first run. Raise it
  later once the split is confirmed working.
- `--host 0.0.0.0` binds the API to all interfaces so your laptop reaches it
  over Tailscale. `--port 8000` is the default; it is explicit here for the
  curl commands in section 6.

Ray places the two ranks one per node because each node exposes exactly one
GPU, so a placement group of two GPUs cannot be satisfied on a single node.
Expected startup output includes lines naming both workers, for example
`RayWorkerVllm` rank 0 and rank 1, each on a different host.

---

## 6. Verify

### 6.1 The model is served

From any machine on the Tailnet (your laptop works):

```bash
curl http://<spark1-ts>:8000/v1/models
```

Expected: a JSON list whose `data` array names the served model, for example
`Qwen/Qwen2.5-7B-Instruct`.

### 6.2 A successful inference response

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

Expected: a `choices` array with an `assistant` message and text back. The
older `/v1/completions` endpoint works the same way with a `prompt` field if
you prefer it; the chat endpoint matches the model's instruct template.

### 6.3 Evidence both nodes participated

A single endpoint answering does not by itself prove the split. Check the
following while the server is running (or during a request):

1. On the head, confirm the placement group is holding both GPUs:

   ```bash
   ray status
   ```

   Expected: `Usage: 2.0/2.0 GPUs` while vLLM is up, with one GPU consumed on
   each node.

2. On both `spark1` and `spark2`, confirm each GPU is loaded and active:

   ```bash
   nvidia-smi
   ```

   Expected: on each node the GB10 shows a large memory reservation (roughly
   half the model weights plus KV cache) and nonzero compute activity during a
   request. Memory use on both nodes at once is the concrete proof the model is
   split, not served twice.

If `nvidia-smi` shows memory on only one node, the second rank did not place;
re-check `RAY_ADDRESS`, that `ray status` lists 2 nodes and 2 GPUs, and that
vLLM is installed on `spark2`.

---

## 7. Caveats and what's next

Multi-node tensor parallelism on GB10 is the least stable part of this stack as
of vLLM 0.28.0 (2026-08-26). Single-node serving on the Spark is a first-class,
validated path; the two-node TP path is newer and less proven. Treat it as
experimental for now and do not assume it is production-ready without testing
your own model and workload.

What is known:

- vLLM officially supports arm64 and targets `sm_121`, and the DGX Spark blog
  documents single-Spark serving as a working, tuned path.
- Two-Spark TP runs have been reported on the NVIDIA forums to hang or drop a
  rank mid-request (one GPU spinning at 100 percent while the peer drops),
  while pipeline parallelism (`--pipeline-parallel-size 2`) on the same pair
  completed. A related thread reports an NCCL allreduce deadlock on dual
  Sparks that affects both vLLM and TensorRT-LLM. These are point-in-time
  reports, not a statement about every release.

Practical fallbacks if TP=2 hangs on your pair:

- Try `--pipeline-parallel-size 2` with `--tensor-parallel-size 1`. Pipeline
  parallelism splits the model by layers instead of by tensor, which the
  reports above found more reliable on two Sparks. It does not keep both GPUs
  as busy as TP, but it does prove a two-node run.
- Re-run the section 03 NCCL test at a large buffer to confirm the fabric is
  still at the high-teens ceiling before blaming vLLM; a regression there is
  more likely MTU 1500 than vLLM.

What's next: later vLLM and CUDA releases are where this improves. The vLLM
release notes already call out Blackwell-specific work (for example the CUDA
graph capture default for Blackwell), and the Spark recipe in NVIDIA's
documentation continues to evolve. Re-verify against the current release and
the DGX Spark forum before treating a specific version as stable.

---

## 8. Expected result

- `ray --version` and `vllm --version` (or `pip show vllm`) report matching,
  compatible versions on both nodes, with vLLM at 0.28.0 (or your pinned later
  release).
- `curl http://<spark1-ts>:8000/v1/models` lists the served model.
- A `/v1/chat/completions` request returns a generated response.
- `ray status` shows `Usage: 2.0/2.0 GPUs` while serving.
- `nvidia-smi` on both nodes shows GPU memory in use, confirming the model is
  split across `spark1` and `spark2`.

To stop, interrupt `vllm serve` (Ctrl-C) on `spark1`; the Ray placement group
releases the GPUs. Tear down Ray afterward with `ray stop` on the worker first,
then the head, as in section 04.
