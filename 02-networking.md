# 02: RoCE Data-Fabric Networking

This section brings up the direct-connect QSFP RoCE link between `spark1` and
`spark2`, sets jumbo frames (MTU 9000), verifies the link and raw RDMA
throughput, and makes the MTU setting survive a reboot.

What you end up with: two nodes, each with both ConnectX-7 logical interfaces
up at MTU 9000 on static RFC 1918 addresses, a clean ping in both directions,
and raw RDMA throughput in the high 190 Gbps range on `ib_write_bw`.

> Prerequisite: `01-os-bringup.md` is done on both nodes (OS installed,
> firmware current, SSH keys in place, Tailscale reachable). The commands below
> run as your regular user with `sudo` where shown.

---

## 1. Two planes, kept apart

Treat the two networks as separate and never mix them in config:

| Plane | Network | Purpose |
|-------|---------|---------|
| Management | Tailscale (out-of-band) | SSH, admin, file transfer, orchestration |
| Data | QSFP RoCE direct-connect | RDMA traffic for NCCL and other collectives |

The direct-connect subnet you create below (`192.168.100.0/24` and `192.168.101.0/24`) is
**not** on Tailscale and **not** routed. It has no default gateway, no DNS, and
no route to the internet. It exists only so the two ConnectX-7 NICs can talk to
each other point-to-point. Do not advertise these subnets into Tailscale and do
not add a default route on them; keep the two planes disjoint so a
misconfigured data plane cannot break your SSH path.

---

## 2. Physical connection

Connect one QSFP cable between the two nodes. Use the same physical port on
both devices (NVIDIA's playbook notes this avoids NCCL test confusion). One
cable is enough for full bandwidth.

---

## 3. Discover the interfaces

The interface names depend on which physical QSFP port you used, so discover
them on each node rather than assuming.

```bash
# Map each RoCE device to its Linux netdev name
ibdev2netdev
```

Expected shape (names are examples):

```
roceP2p1s0f0 port 1 ==> enP2p1s0f0np0 (Up)
roceP2p1s0f1 port 1 ==> enP2p1s0f1np1 (Down)
rocep1s0f0  port 1 ==> enp1s0f0np0  (Up)
rocep1s0f1  port 1 ==> enp1s0f1np1  (Down)
```

Notes:

- The two NICs use different capitalization in their names: `rocep1s0f0`
  (lowercase `p1`) vs `roceP2p1s0f0` (uppercase `P2`), and the same pattern
  holds for the netdev names (`enp1s0f0np0` vs `enP2p1s0f0np0`). Copy the
  names from your own `ibdev2netdev` output, not from these examples.
- `f0`/`f1` is the port. If you used the other QSFP port, you will see the
  `f1` pair Up instead of `f0`. Use whichever pair reports `(Up)`.

Confirm link state and rate for each RoCE device:

```bash
# Replace with your RoCE device names from ibdev2netdev
ibstat rocep1s0f0
ibstat roceP2p1s0f0
```

You want `LinkUp` and `Rate: 200` on each. If a device shows `Down`, reseat the
cable, then reboot the nodes and re-check; NVIDIA's playbook reports that a
reboot clears a stuck down link.

Finally, list the interfaces and note the current MTU (it starts at 1500):

```bash
ip link show enp1s0f0np0
ip link show enP2p1s0f0np0
```

---

## 4. Why one cable lights up two interfaces

Each physical QSFP port appears as two logical interfaces, one per ConnectX-7
controller, because the two NICs are bridged into a single L2 domain by the
on-die eSwitch. When you plug one cable into a port, carrier comes Up on both
logical interfaces for that port at once.

So seeing both `enp1s0f0np0` and `enP2p1s0f0np0` report `(Up)` after one cable
is expected, not evidence of a second link or a split port. They are two
representations of the same physical port. This is also why NCCL can stripe
channels across both NICs over a single cable: the two controllers share the
same wire through the eSwitch.

For full bandwidth, assign IPs to both Up interfaces (one subnet per NIC), as
the next step does.

---

## 5. Assign static IPs (netplan)

DGX Spark ships an Ubuntu 24.04 LTS based OS (NVIDIA forum reports 24.04.3).
Ubuntu configures networking through netplan (YAML). Two renderers are
available:

- **NetworkManager** is the default renderer on this Ubuntu Desktop image
  (the `ubuntu-settings` package ships
  `/usr/lib/netplan/00-network-manager-all.yaml` with
  `renderer: NetworkManager`).
- **systemd-networkd** (`networkd`) is the default on Ubuntu Server and is a
  valid alternative; force it with `renderer: networkd` in your file.

You can confirm which is active before you start:

```bash
ls /etc/netplan/                 # see existing files
cat /usr/lib/netplan/00-network-manager-all.yaml   # desktop default marker
```

The example below omits the `renderer` key, so it inherits whatever the box
already uses. NVIDIA's own performance guide for the Connect Two Sparks
playbook sets `renderer: NetworkManager` explicitly in this same file; include
that line if you want to pin it.

The direct-connect link is point-to-point: static RFC 1918 addresses, no
DHCP, no gateway. Use one `/24` per NIC so the two NICs sit on distinct
subnets and NCCL has clean routes to stripe across.

On **spark1**:

```bash
sudo tee /etc/netplan/40-cx7.yaml > /dev/null <<'EOF'
network:
  version: 2
  ethernets:
    enp1s0f0np0:
      dhcp4: no
      mtu: 9000
      addresses:
        - 192.168.100.10/24
    enP2p1s0f0np0:
      dhcp4: no
      mtu: 9000
      addresses:
        - 192.168.101.10/24
EOF
sudo chmod 600 /etc/netplan/40-cx7.yaml
sudo netplan apply
```

On **spark2**:

```bash
sudo tee /etc/netplan/40-cx7.yaml > /dev/null <<'EOF'
network:
  version: 2
  ethernets:
    enp1s0f0np0:
      dhcp4: no
      mtu: 9000
      addresses:
        - 192.168.100.11/24
    enP2p1s0f0np0:
      dhcp4: no
      mtu: 9000
      addresses:
        - 192.168.101.11/24
EOF
sudo chmod 600 /etc/netplan/40-cx7.yaml
sudo netplan apply
```

If you only need a minimal single-interface link, configure just the first
NIC pair on `192.168.100.10/24` and `192.168.100.11/24` and skip the second subnet.

Notes:

- The `mtu: 9000` line in the same file is what persists the jumbo frame
  setting across reboots (see the next section). You can add MTU and addresses
  in one pass, as above.
- If `netplan apply` errors with `Invalid YAML: aliases are not supported` and
  points at a `90-NM-*.yaml` file, that file may be corrupted (a known Spark
  issue). Back it up and replace it with a minimal valid stub, or rename it to
  `.disabled`, then re-apply. Do not let it silently break your data plane.

---

## 6. MTU 9000, persisted

Jumbo frames are mandatory on the direct-connect interfaces for full RoCE
throughput. At MTU 1500, NCCL sits around 3.0 GB/s even though the link looks
fine; raising both sides to 9000 restores the high-teens GB/s ceiling.

The `mtu: 9000` lines above persist the setting. To change it immediately
without touching netplan (or to fix a freshly booted node in a pinch):

```bash
sudo ip link set dev enp1s0f0np0 mtu 9000
sudo ip link set dev enP2p1s0f0np0 mtu 9000
```

Run that on **both** nodes, on matching interfaces.

Two things to watch:

- The `ip link set` form is ephemeral. It does not survive a reboot. The
  netplan `mtu: 9000` is the durable fix.
- Firmware updates have been known to silently reset MTU back to 1500. After
  any firmware update, and after any reboot, re-verify the MTU (next section)
  and re-apply if it has reverted.

---

## 7. Verify

Run these in order, on both nodes.

1. Confirm the MTU stuck at 9000 on every direct-connect interface:

   ```bash
   ip link show enp1s0f0np0
   ip link show enP2p1s0f0np0
   ```

   Look for `mtu 9000` and `state UP` on each.

2. Ping across the link in both directions. From `spark1`:

   ```bash
   ping -c 3 192.168.100.11
   ping -c 3 192.168.101.11
   ```

   And from `spark2`, ping `192.168.100.10` and `192.168.101.10`. Any packet loss here
   means the cable, a subnet, or the MTU is wrong before you get to RDMA.

3. Raw RDMA throughput with `ib_write_bw` (from the `perftest` package). The
   flags below are from current perftest (`linux-rdma/perftest`, man page as
   of 2026-09-03):

   On **spark2** (server):

   ```bash
   ib_write_bw -d rocep1s0f0 -a --report_gbits
   ```

   On **spark1** (client):

   ```bash
   ib_write_bw -d rocep1s0f0 192.168.100.11 -a --report_gbits
   ```

   Flag meanings: `-d <dev>` selects the RoCE device; `-a` runs all message
   sizes from 2 bytes up to 2^23 (8 MiB); `--report_gbits` reports bandwidth
   in Gbit/s instead of MiB/s. The control channel defaults to TCP port 18515
   (add `-p <port>` only if that port is blocked, which it normally is not on
   a direct link). For RoCE the driver resolves the IPv4 GID automatically;
   if the client cannot find the peer, add the GID index flag `-x 3` (the
   usual RoCEv2 IPv4 index).

   Expected result: roughly 100+ Gbps from 64 KB upward, peaking around 196
   Gbps at the largest sizes. That is the verified raw RDMA ceiling for this
   link.

   Repeat against the second NIC pair (`roceP2p1s0f0`, peer `192.168.101.11`) to
   confirm both NICs carry traffic.

   One caveat: a `syndrom 0x88` crash at the 128 KB message size in
   `ib_write_bw` is a known, test-harness-specific artifact (single-QP
   reliable connection). It is not a fault in your fabric. NCCL's real data
   path is unaffected, and the crash does not mean the link is broken.

---

## 8. Expected result

- `ibdev2netdev` shows two `(Up)` interfaces per node.
- `ibstat` reports `LinkUp` and `Rate: 200` on both RoCE devices.
- `ip link show` reports `mtu 9000` and `state UP` on both interfaces.
- `ping` succeeds in both directions on both subnets.
- `ib_write_bw` shows roughly 100+ Gbps up to 64 KB and near 196 Gbps at the
  largest sizes, on both NIC pairs.

If MTU reads 1500, NCCL will cap around 3 GB/s later (see `03-nccl.md`), so do
not skip the re-check after any reboot or firmware update.
