# 05: Docker + NVIDIA Container Toolkit

This section installs Docker Engine and the NVIDIA Container Toolkit on both
nodes, wires Docker to the `nvidia` runtime so containers can see the GB10
Blackwell GPU, and verifies GPU access from inside a container.

What you end up with: both nodes running Docker with a working `nvidia`
runtime, and a single command that prints the Blackwell GPU in `nvidia-smi`
from inside a container on each node.

> Prerequisite: `01-os-bringup.md` is done on both nodes (OS installed, NVIDIA
> driver and CUDA 13 present, operator user with sudo). The commands below run
> as your regular user with `sudo` where shown.

---

## 0. Reference versions (verified)

Verified against current official docs as of 2026-09-03.

| Component | Version / source |
| --- | --- |
| NVIDIA Container Toolkit | 1.20.0 (current release line) |
| Docker Engine | apt repo, Ubuntu 24.04 (noble), arm64 |
| CUDA base image (verify step) | `nvidia/cuda:13.0.3-runtime-ubuntu24.04` (arm64 multi-arch) |

Sources:

- Toolkit install guide: https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html
- Toolkit sample workload: https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/sample-workload.html
- Docker install (Ubuntu): https://docs.docker.com/engine/install/ubuntu/

The exact point releases change over time; confirm against those pages rather
than treating the table as fixed.

---

## 1. Install Docker (both nodes)

DGX OS 7.5.0 is Ubuntu 24.04 (noble) based, and the Spark is arm64 (aarch64).
Docker's Ubuntu apt repository is multi-arch and publishes arm64, so the
standard per-distro path works unchanged. The `Architectures` line below uses
`dpkg --print-architecture`, which returns `arm64` on these nodes, so no
special architecture handling is needed.

Run on each node:

```bash
# Add Docker's official GPG key
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources
sudo tee /etc/apt/sources.list.d/docker.sources > /dev/null <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update

# Install Docker Engine, CLI, containerd, and the Compose and Buildx plugins
sudo apt install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
```

Notes:

- The `Suites` line resolves the Ubuntu codename from `/etc/os-release`. On a
  stock Ubuntu 24.04 image this is `noble`. If a derivative `/etc/os-release`
  does not provide an Ubuntu codename and the line comes out empty, hardcode
  `noble` (Ubuntu 24.04) instead of leaving it blank.
- Confirm the daemon is running: `sudo systemctl status docker` (if not,
  `sudo systemctl enable --now docker`).
- Optional but convenient: add your operator user to the `docker` group so you
  can run `docker` without `sudo`. `sudo usermod -aG docker $USER`, then log
  out and back in. This is not required for the verification below, which uses
  `sudo`.

Sanity check that Docker works at all before moving on:

```bash
sudo docker run --rm hello-world
```

Expected: the "Hello from Docker!" message. This pulls an arm64 `hello-world`
image and confirms the daemon, the registry, and the network path all work.

---

## 2. Install the NVIDIA Container Toolkit (both nodes)

The toolkit is what bridges Docker to the host NVIDIA driver and the GPU.
Install it from NVIDIA's apt repository:

```bash
# Prerequisites
sudo apt-get update
sudo apt-get install -y --no-install-recommends ca-certificates curl gnupg2

# Configure NVIDIA's production repository
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey \
  | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list \
  | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' \
  | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt-get update

# Install the toolkit packages
export NVIDIA_CONTAINER_TOOLKIT_VERSION=1.20.0-1
sudo apt-get install -y \
  nvidia-container-toolkit=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
  nvidia-container-toolkit-base=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
  libnvidia-container-tools=${NVIDIA_CONTAINER_TOOLKIT_VERSION} \
  libnvidia-container1=${NVIDIA_CONTAINER_TOOLKIT_VERSION}
```

Notes:

- The toolkit's apt repository is multi-arch, so this path works on arm64
  without any special handling.
- The four-package list is the documented install. If you prefer to track the
  latest patch release, install `nvidia-container-toolkit` without the version
  pin and let apt pull its dependencies; pinning is shown here so both nodes
  end up on an identical version.
- Do not install the older `nvidia-docker2` package. It is the legacy path;
  the current toolkit replaces it.

---

## 3. Configure the `nvidia` runtime

Point Docker at the toolkit's runtime, then restart the daemon:

```bash
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

What this does:

- `nvidia-ctk runtime configure --runtime=docker` edits
  `/etc/docker/daemon.json` and registers an `nvidia` runtime whose `path` is
  `nvidia-container-runtime`. It does not set the `nvidia` runtime as the
  Docker default; the default runtime remains `runc`.
- Because the runtime is registered but not the default, you select it
  per-container. The modern, recommended way is a device request: `--gpus
  all`. Docker turns `--gpus all` into a request for the `gpu` capability,
  which the toolkit's runtime handles by exposing every GPU to the container
  (equivalent to `NVIDIA_VISIBLE_DEVICES=all`). With one GPU per node,
  `--gpus all` and `--gpus 1` mean the same thing.
- The explicit form `--runtime=nvidia` also works and forces the `nvidia`
  runtime regardless of device requests. NVIDIA's own sample workload passes
  both flags (`--runtime=nvidia --gpus all`), which is redundant but
  unambiguous; `--gpus all` alone is enough with a correctly configured
  daemon.

Confirm the runtime is registered:

```bash
sudo docker info | grep -A2 Runtimes
```

Expected output names both `runc` and `nvidia`, for example:

```
 Runtimes: nvidia runc
```

If `nvidia` is missing, re-run the `nvidia-ctk` command above and restart the
daemon before proceeding.

---

## 4. Verify GPU access inside a container (both nodes)

The canonical check is NVIDIA's sample workload: run `nvidia-smi` from inside
a plain container. The binary is not baked into the image; the `nvidia`
runtime injects the host's `nvidia-smi` (and the driver libraries) into the
container at run time, so the GPU pass-through is what you are testing.

```bash
sudo docker run --rm --runtime=nvidia --gpus all ubuntu nvidia-smi
```

Expected output includes the Blackwell GPU line, for example:

```
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 580.159.03    Driver Version: 580.159.03    CUDA Version: 13.0    |
|-------------------------------+----------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id        Disp.A | Volatile |
|   0  NVIDIA GB10              On        | 00000000:00:1E.0 Off |      N/A |
+-------------------------------+----------------------+----------------------+
```

The key facts to confirm: the driver version matches the host, `CUDA Version`
is 13.0 (the shipped level), and the GPU name is `NVIDIA GB10`. If you see
`NVIDIA-SMI has failed because it couldn't communicate with the NVIDIA driver`,
the runtime is not passing the GPU through; go back to section 3 and re-check
`docker info` and the daemon restart.

### Exact CUDA image tag for arm64

For containers that actually need CUDA tooling (Ray, vLLM, custom builds),
pull the CUDA image from Docker Hub, not NGC, and use the Ubuntu 24.04 base:

```bash
sudo docker run --rm --gpus all \
  nvidia/cuda:13.0.3-runtime-ubuntu24.04 \
  nvidia-smi
```

Two arm64-specific gotchas on the Spark:

- Use `docker.io/nvidia/cuda` (Docker Hub), not `nvcr.io/nvidia/cuda` (NGC).
  The NGC CUDA images do not publish arm64 tags; the Docker Hub `nvidia/cuda`
  images are multi-arch and include arm64. This is a reported DGX Spark
  gotcha, not a general container fact.
- Use the `ubuntu24.04` tag, not `ubuntu22.04`. DGX OS ships glibc 2.38;
  Ubuntu 22.04 containers carry glibc 2.35 and the host-injected driver
  binaries can fail with `version GLIBC_2.38 not found`.

The `13.0.3` patch matches the shipped CUDA 13.0 series. Newer 13.x tags are
also published multi-arch; the constraint is only that the container's CUDA
must be no newer than what the host driver (580.159.03) supports. For a base
to build against, substitute the `devel` variant
(`nvidia/cuda:13.0.3-devel-ubuntu24.04`), which adds nvcc and headers.

---

## 5. Container networking to the RoCE data fabric (optional, advanced)

The default Docker bridge network does not carry RDMA, so a container that
needs to talk to the direct-connect fabric (for example a Ray or vLLM worker
doing collective traffic over RoCE) must be placed on the host network or
given explicit access to the RoCE interfaces.

The simplest path is host networking, which exposes the host's RoCE
interfaces and their `192.168.100.x` / `192.168.101.x` addresses directly:

```bash
sudo docker run --rm --network host \
  --gpus all \
  nvidia/cuda:13.0.3-runtime-ubuntu24.04 \
  ip -br addr show enp1s0f0np0
```

With `--network host` the container sees the same `enp1s0f0np0` /
`enP2p1s0f0np0` interfaces (and MTU 9000) that section 02 configured. For
actual RDMA traffic you also need the RDMA devices and the verbs libraries
visible inside the container, typically via `--device=/dev/infiniband/...`
entries (or `--privileged`) plus `libibverbs` installed in the image, and the
usual NCCL env vars (`NCCL_IB_HCA`, `NCCL_SOCKET_IFNAME`) pointed at those
interfaces. This is covered in section 06; treat it as advanced and revisit
only if Ray or vLLM will run inside containers rather than directly on the
host.

If host networking is too broad, the alternative is an ipvlan or macvlan
network attached to the specific RoCE interface, but that adds setup for no
throughput benefit on a two-node direct link, so host networking is the
pragmatic choice here.

---

## 6. Expected result

Both nodes are at the same state when you can answer yes to all of these:

- `sudo docker run --rm hello-world` prints "Hello from Docker!" on each node.
- `sudo docker info | grep -A2 Runtimes` lists both `runc` and `nvidia`.
- `sudo docker run --rm --runtime=nvidia --gpus all ubuntu nvidia-smi` prints
  the NVIDIA GB10 GPU with `CUDA Version: 13.0` on each node.

Repeat the verification on both `spark1` and `spark2`; a single-node pass does
not prove the pair is ready. Once both print the GPU, continue to `04-ray.md`
and `06-vllm.md`, which build on this runtime.
