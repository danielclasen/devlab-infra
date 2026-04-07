# devlab Networking

Complete reference for the devlab-rack01 network stack: physical cabling, switch configuration, Talos node bonding/VLAN setup, and Kubernetes RoCEv2/RDMA integration.

---

## Table of Contents

1. [VLAN Architecture](#vlan-architecture)
2. [Physical Topology](#physical-topology)
3. [sw01 — MikroTik CRS504-4XQ-IN (Storage/RoCE)](#sw01--mikrotik-crs504-4xq-in-storageroce)
4. [sw02 — MikroTik CRS312-4C+8XG (Control Plane)](#sw02--mikrotik-crs312-4c8xg-control-plane)
5. [Talos Node Configuration](#talos-node-configuration)
6. [Kernel Modules & Sysctls](#kernel-modules--sysctls)
7. [Kubernetes RDMA Setup](#kubernetes-rdma-setup)
8. [NIC-side QoS (PFC + ETS)](#nic-side-qos-pfc--ets)
9. [dmf-mxl Workload Requirements](#dmf-mxl-workload-requirements)
10. [Rebuild Checklist](#rebuild-checklist)
11. [Pending / Follow-up](#pending--follow-up)

---

## VLAN Architecture

| VLAN | Name | Subnet | MTU | Gateway | Notes |
|------|------|--------|-----|---------|-------|
| 200 | LAB_MANAGE | 10.200.0.0/21 | 1500 | 10.200.0.1 | OOB management — switches, BMCs |
| 290 | LAB_CPL | 10.200.90.0/24 | 1500 | 10.200.90.1 | Kubernetes control plane / Talos API |
| 291 | LAB_STOR | 10.200.91.0/24 | 9000 | none (L2 only) | Linstor/DRBD storage replication |
| 292 | LAB_ROCE | 10.200.92.0/24 | 9000 | none (L2 only) | RDMA over Converged Ethernet |
| 299 | LAB_BGP | 10.200.99.0/24 | 1500 | 10.200.99.1 | MetalLB BGP peering with OPNsense |

**Node address scheme (rack01):**

| Node | VLAN 290 (CPL) | VLAN 291 (STOR) | VLAN 292 (RoCE) | VLAN 299 (BGP) |
|------|---------------|----------------|----------------|---------------|
| n01 | 10.200.90.111 | 10.200.91.11 | 10.200.92.11 | 10.200.99.11 |
| n02 | 10.200.90.112 | 10.200.91.12 | 10.200.92.12 | 10.200.99.12 |
| n03 | 10.200.90.113 | 10.200.91.13 | 10.200.92.13 | 10.200.99.13 |
| n04 | 10.200.90.114 | 10.200.91.14 | 10.200.92.14 | 10.200.99.14 |

Kubernetes API VIP: `10.200.90.10` (shared across all 4 nodes via Talos VIP)

---

## Physical Topology

### Hardware

| Device | Model | Role |
|--------|-------|------|
| sw01 | MikroTik CRS504-4XQ-IN | Storage/RoCE switch (VLANs 291, 292) |
| sw02 | MikroTik CRS312-4C+8XG | Control-plane switch (VLANs 290, 299) |
| n01–n04 | (rack01 compute nodes) | Talos Linux, Kubernetes control-plane |

### sw01 Port Layout (Storage/RoCE — CRS504-4XQ-IN)

The CRS504-4XQ-IN has 4× QSFP28 100G ports. Each is used with a **4×25G breakout cable**, giving 16 physical 25G lanes total.

```
QSFP28-1 (100G) ──┬── lane 1 (qsfp28-1-1, 25G) ─── n01 ens2f0np0  ┐
                  ├── lane 2 (qsfp28-1-2, 25G) ─── n02 ens2f0np0  │ bond1
                  ├── lane 3 (qsfp28-1-3, 25G) ─── n03 ens2f0np0  │ (VLANs
                  └── lane 4 (qsfp28-1-4, 25G) ─── n04 ens2f0np0  │  291+292)

QSFP28-2 (100G) ──┬── lane 1 (qsfp28-2-1, 25G) ─── n01 ens2f1np1 ┘
                  ├── lane 2 (qsfp28-2-2, 25G) ─── n02 ens2f1np1
                  ├── lane 3 (qsfp28-2-3, 25G) ─── n03 ens2f1np1
                  └── lane 4 (qsfp28-2-4, 25G) ─── n04 ens2f1np1
```

Each node gets **2×25G = 50 Gbps** in an LACP bond (`bonding1nXX` on sw01, `bond1` on the host).

### sw02 Port Layout (Control Plane — CRS312-4C+8XG)

The CRS312-4C+8XG has 8× 10G RJ45 + 4× SFP+/RJ45 combo ports.

```
ether1+ether2 ─── n01 eno3+eno4 (10G) ┐
ether3+ether4 ─── n02 eno3+eno4 (10G) │ bond0
ether5+ether6 ─── n03 eno1+eno2  (1G) │ (VLAN 290 native, VLAN 299 tagged)
ether7+ether8 ─── n04 eno1+eno2  (1G) ┘

combo1+combo2 ─── uplink LAG (bonding1) → core switch / OPNsense
combo3        ─── secondary uplink (tagged: VLANs 200, 290, 299)
```

Each node gets a 2-port LACP bond (`bonding1nXX` on sw02, `bond0` on the host). VLAN 290 is the **native/untagged** VLAN on node-facing ports (pvid=290); VLAN 299 is tagged.

### Node NIC Layout

| Node | bond1 port A → sw01 | bond1 port B → sw01 | bond0 ports → sw02 | bond0 speed |
|------|---------------------|---------------------|---------------------|-------------|
| n01 | ens2f0np0 → qsfp28-1-1 | ens2f1np1 → qsfp28-2-1 | eno3+eno4 → ether1+ether2 | 2×10G |
| n02 | ens2f0np0 → qsfp28-1-2 | ens2f1np1 → qsfp28-2-2 | eno3+eno4 → ether3+ether4 | 2×10G |
| n03 | ens2f0np0 → qsfp28-1-3 | ens2f1np1 → qsfp28-2-3 | eno1+eno2 → ether5+ether6 | 2×1G |
| n04 | ens2f0np0 → qsfp28-1-4 | ens2f1np1 → qsfp28-2-4 | eno1+eno2 → ether7+ether8 | 2×1G |

> **Note:** PCI bus addresses for the CX4 Lx differ between nodes (0000:05:00.x on n01/n02, 0000:04:00.x on n03/n04), but the NIC interface names are consistent (`ens2f0np0` / `ens2f1np1`) across all four nodes due to predictable NIC naming.

---

## sw01 — MikroTik CRS504-4XQ-IN (Storage/RoCE)

**Model:** CRS504-4XQ-IN  
**Switch chip:** Marvell-98DX4310  
**RouterOS:** 7.22.1 (stable)  
**Management IP:** 10.200.1.1/21 (VLAN 200 OOB)

### Bridge Layout

| Bridge | Purpose | VLAN filtering | STP | MTU |
|--------|---------|---------------|-----|-----|
| bridge | Management (VLAN 200) + uplinks | yes | RSTP | 1500 |
| bridge1 | Storage (VLAN 291) + RoCE (VLAN 292) | yes | **none** | 9000 |

`bridge1` has STP disabled (`protocol-mode=none`) because all 4 server bonds are leaf connections with no redundant L2 paths. RSTP topology change notifications would cause unnecessary MAC table flushes on bond failover events.

### Complete RouterOS Configuration

The following is a full `/export hide-sensitive` that can be applied to restore the switch from scratch:

```routeros
# MikroTik CRS504-4XQ-IN — devlab-rack01 sw01
# RouterOS 7.22.1

/interface bridge
add admin-mac=<mgmt-mac> auto-mac=no comment=defconf name=bridge vlan-filtering=yes
add mtu=9000 name=bridge1 protocol-mode=none vlan-filtering=yes

/interface ethernet
set [ find default-name=ether1 ] name=mgmt
set [ find default-name=qsfp28-1-1 ] l2mtu=9084 mtu=9000
set [ find default-name=qsfp28-1-2 ] l2mtu=9084 mtu=9000
set [ find default-name=qsfp28-1-3 ] l2mtu=9084 mtu=9000
set [ find default-name=qsfp28-1-4 ] l2mtu=9084 mtu=9000
set [ find default-name=qsfp28-2-1 ] l2mtu=9084 mtu=9000
set [ find default-name=qsfp28-2-2 ] l2mtu=9084 mtu=9000
set [ find default-name=qsfp28-2-3 ] l2mtu=9084 mtu=9000
set [ find default-name=qsfp28-2-4 ] l2mtu=9084 mtu=9000

/interface vlan
add interface=bridge  name=200_LAB_MGM  vlan-id=200
add interface=bridge1 name=291_LAB_STOR vlan-id=291 mtu=9000
add interface=bridge1 name=292_LAB_ROCE vlan-id=292 mtu=9000

/interface bonding
add mode=802.3ad lacp-rate=1sec mtu=9000 name=bonding1n01 \
    slaves=qsfp28-1-1,qsfp28-2-1 transmit-hash-policy=layer-3-and-4
add mode=802.3ad lacp-rate=1sec mtu=9000 name=bonding1n02 \
    slaves=qsfp28-1-2,qsfp28-2-2 transmit-hash-policy=layer-3-and-4
add mode=802.3ad lacp-rate=1sec mtu=9000 name=bonding1n03 \
    slaves=qsfp28-1-3,qsfp28-2-3 transmit-hash-policy=layer-3-and-4
add mode=802.3ad lacp-rate=1sec mtu=9000 name=bonding1n04 \
    slaves=qsfp28-1-4,qsfp28-2-4 transmit-hash-policy=layer-3-and-4

/interface bridge port
add bridge=bridge1 interface=bonding1n01
add bridge=bridge1 interface=bonding1n02
add bridge=bridge1 interface=bonding1n03
add bridge=bridge1 interface=bonding1n04

/interface bridge vlan
add bridge=bridge1 vlan-ids=291 comment=291_LAB_STOR \
    tagged=bridge1,bonding1n01,bonding1n02,bonding1n03,bonding1n04
add bridge=bridge1 vlan-ids=292 comment=292_LAB_ROCE \
    tagged=bridge1,bonding1n01,bonding1n02,bonding1n03,bonding1n04

/interface ethernet switch
set switch1 l3-hw-offloading=yes qos-hw-offloading=yes

# --- RoCEv2 QoS ---

/interface ethernet switch qos profile
add name=roce traffic-class=3
add name=cnp  traffic-class=6

/interface ethernet switch qos map ip
add dscp=26 profile=roce
add dscp=48 profile=cnp

/interface ethernet switch qos tx-manager queue
set 1 schedule=high-priority-group weight=1
set 3 schedule=high-priority-group weight=1 shared-pool-index=1 ecn=yes
set 6 schedule=strict-priority

/interface ethernet switch qos priority-flow-control
add name=pfc-tc3 traffic-class=3 rx=yes tx=yes

/interface ethernet switch qos port
set qsfp28-1-1 trust-l3=keep pfc=pfc-tc3 egress-rate-queue3=25Gbps
set qsfp28-1-2 trust-l3=keep pfc=pfc-tc3 egress-rate-queue3=25Gbps
set qsfp28-1-3 trust-l3=keep pfc=pfc-tc3 egress-rate-queue3=25Gbps
set qsfp28-1-4 trust-l3=keep pfc=pfc-tc3 egress-rate-queue3=25Gbps
set qsfp28-2-1 trust-l3=keep pfc=pfc-tc3 egress-rate-queue3=25Gbps
set qsfp28-2-2 trust-l3=keep pfc=pfc-tc3 egress-rate-queue3=25Gbps
set qsfp28-2-3 trust-l3=keep pfc=pfc-tc3 egress-rate-queue3=25Gbps
set qsfp28-2-4 trust-l3=keep pfc=pfc-tc3 egress-rate-queue3=25Gbps

/ip neighbor discovery-settings set lldp-dcbx=yes

/ip address
add address=10.200.1.1/21 comment=oob-mgmt interface=mgmt network=10.200.0.0

/ip route
add dst-address=0.0.0.0/0 gateway=10.200.0.1

/ip dns
set servers=10.200.0.1

/system clock
set time-zone-name=Europe/Berlin

/system identity
set name="MikroTik CRS504-4XQ-IN"
```

### RoCEv2 QoS Design

> **Verified live on RouterOS 7.22.1** — all 8 node-facing ports confirmed via `/interface ethernet switch qos port print terse`.

| Traffic class | Profile | DSCP | Queue | Scheduler | Notes |
|--------------|---------|------|-------|-----------|-------|
| Best-effort | default | 0 | TC1 | ETS weight=1 | Storage (DRBD/Linstor), general traffic |
| RoCEv2 data | roce | 26 | TC3 | ETS weight=1, lossless pool, ECN | RDMA data path |
| CNP | cnp | 48 | TC6 | Strict priority | Congestion Notification Packets |

- **PFC** (IEEE 802.1Qbb) enabled on TC3 with `rx=yes tx=yes` — lossless for RDMA traffic
- **ECN** enabled on TC3 queue — switch marks CE bits instead of dropping at threshold
- **ETS** — TC1 and TC3 share bandwidth equally when both are active; TC6 always drains first
- **LLDP DCBX** enabled — switch advertises QoS capabilities to Mellanox NICs
- `trust-l3=keep` — switch honors host-set DSCP values without overwriting them
- `egress-rate-queue3=25Gbps` — per-port TC3 egress rate cap matches link speed

---

## sw02 — MikroTik CRS312-4C+8XG (Control Plane)

**Model:** CRS312-4C+8XG  
**Switch chip:** Marvell-98DX3257 (88E series)  
**RouterOS:** 7.21.3 (stable)  
**Management IP:** 10.200.1.2/21 (VLAN 200 OOB)

### Bridge Layout

| Bridge | Purpose | VLAN filtering | STP | Notes |
|--------|---------|---------------|-----|-------|
| bridge | All VLANs (200, 290, 299) | yes | RSTP | Single bridge — node ports, uplinks |

Node-facing bond ports use `pvid=290` so VLAN 290 is untagged toward the nodes. VLAN 299 is tagged on all node bonds (for MetalLB BGP sub-interface `bond0.299`).

### VLAN Assignment on sw02

| VLAN | Tagged on | Untagged on | Purpose |
|------|-----------|-------------|---------|
| 200 | combo3, bonding1, bridge | — | OOB management |
| 290 | bridge, bonding1, combo3 | bonding1n01–n04 (pvid=290) | K8s control plane (native VLAN) |
| 299 | bridge, bonding1, combo3, bonding1n01–n04 | — | MetalLB BGP peering |

### Complete RouterOS Configuration

```routeros
# MikroTik CRS312-4C+8XG — devlab-rack01 sw02
# RouterOS 7.21.3

/interface bridge
add admin-mac=<mgmt-mac> auto-mac=no comment=defconf \
    ingress-filtering=no name=bridge port-cost-mode=short vlan-filtering=yes

/interface ethernet
set [ find default-name=ether9 ] name=mgmt

/interface vlan
add interface=bridge name=200_LAB_MGM  vlan-id=200
add interface=bridge name=290_LAB_CPL  vlan-id=290
add interface=bridge name=299_LAB_BGP  vlan-id=299

/interface bonding
add mode=802.3ad lacp-rate=1sec name=bonding1 \
    slaves=combo1,combo2 transmit-hash-policy=layer-2-and-3
add mode=802.3ad lacp-rate=1sec name=bonding1n01 \
    slaves=ether1,ether2 transmit-hash-policy=layer-3-and-4
add mode=802.3ad lacp-rate=1sec name=bonding1n02 \
    slaves=ether3,ether4 transmit-hash-policy=layer-3-and-4
add mode=802.3ad lacp-rate=1sec name=bonding1n03 \
    slaves=ether5,ether6 transmit-hash-policy=layer-3-and-4
add mode=802.3ad lacp-rate=1sec name=bonding1n04 \
    slaves=ether7,ether8 transmit-hash-policy=layer-3-and-4

/interface bridge port
add bridge=bridge frame-types=admit-only-vlan-tagged interface=combo3 \
    internal-path-cost=10 path-cost=10
add bridge=bridge frame-types=admit-only-vlan-tagged interface=combo4 \
    internal-path-cost=10 path-cost=10
add bridge=bridge frame-types=admit-only-vlan-tagged interface=bonding1 \
    internal-path-cost=10 path-cost=10
add bridge=bridge interface=bonding1n01 pvid=290
add bridge=bridge interface=bonding1n02 pvid=290
add bridge=bridge interface=bonding1n03 pvid=290
add bridge=bridge interface=bonding1n04 pvid=290

/interface bridge vlan
add bridge=bridge vlan-ids=200 comment=200_LAB_MGT \
    tagged=combo3,bonding1,bridge
add bridge=bridge vlan-ids=290 comment=290_LAB_CPL \
    tagged=bridge,bonding1,combo3 \
    untagged=bonding1n01,bonding1n02,bonding1n03,bonding1n04
add bridge=bridge vlan-ids=299 comment=299_LAB_BGP \
    tagged=bridge,bonding1,combo3,bonding1n01,bonding1n02,bonding1n03,bonding1n04

/ip firewall connection tracking
set udp-timeout=10s

/ip settings
set max-neighbor-entries=8192

/ipv6 settings
set disable-ipv6=yes max-neighbor-entries=8192

/ip address
add address=10.200.1.2/21 comment=oob-mgmt interface=mgmt network=10.200.0.0

/ip route
add dst-address=0.0.0.0/0 gateway=10.200.0.1

/ip dns
set servers=10.200.0.1

/routing bfd configuration
add interfaces=all min-rx=200ms min-tx=200ms multiplier=5

/system clock
set time-zone-name=Europe/Berlin

/system ntp client
set enabled=yes
/system ntp client servers
add address=10.200.0.1

/system identity
set name="Mikrotik CRS312-4C+8XG"
```

---

## Talos Node Configuration

Configuration lives in `talos/talconfig.yaml` and is rendered via `talhelper genconfig`.

### Bond Interfaces per Node

Each node has two bonds:

**bond0 — Control plane (VLAN 290 native + VLAN 299 tagged)**

| Node | Interfaces | Speed | DHCP (VLAN 290) | VLAN 299 static |
|------|-----------|-------|-----------------|-----------------|
| n01, n02 | eno3 + eno4 | 10G | yes | 10.200.99.11/24, .12/24 |
| n03, n04 | eno1 + eno2 | 1G | yes | 10.200.99.13/24, .14/24 |

Mode: 802.3ad, lacpRate: fast, xmitHashPolicy: layer3+4, MTU: 1500

**bond1 — Storage + RoCE (VLAN 291 + VLAN 292 tagged)**

| Node | Interfaces | Speed | VLAN 291 static | VLAN 292 static |
|------|-----------|-------|-----------------|-----------------|
| all | ens2f0np0 + ens2f1np1 | 2×25G | 10.200.91.{11-14}/24 | 10.200.92.{11-14}/24 |

Mode: 802.3ad, lacpRate: fast, xmitHashPolicy: layer3+4, MTU: 9000 (jumbo)

> bond1 carries no IP on the bond interface itself — only on the VLAN sub-interfaces.

---

## Kernel Modules & Sysctls

### Kernel Modules (loaded via `talconfig.yaml` `controlPlane.kernelModules`)

| Module | Purpose |
|--------|---------|
| `drbd` | DRBD block replication (`usermode_helper=disabled`) |
| `drbd_transport_tcp` | DRBD TCP transport |
| `dm-thin-pool` | LVM thin provisioning (Linstor backing) |
| `mlx5_ib` | Mellanox CX4 Lx InfiniBand/RDMA driver |
| `ib_uverbs` | User-space RDMA verbs interface |
| `ib_umad` | User-space MAD interface — creates `/dev/infiniband/umad0` |
| `rdma_ucm` | User-space connection manager — creates `/dev/infiniband/rdma_cm` |

> **`rdma_cm` does not exist as a loadable module** in the Talos 6.18.18 kernel. The correct module is `rdma_ucm` which provides the same userspace `/dev/infiniband/rdma_cm` char device.

### Sysctls (applied via `talos/patches/controlPlane/001-roce-sysctls.yaml`)

| Sysctl | Value | Purpose |
|--------|-------|---------|
| `net.core.rmem_max` | 67108864 (64 MiB) | Max socket receive buffer |
| `net.core.wmem_max` | 67108864 (64 MiB) | Max socket send buffer |
| `net.core.rmem_default` | 67108864 | Default socket receive buffer |
| `net.core.wmem_default` | 67108864 | Default socket send buffer |
| `net.ipv4.conf.all.rp_filter` | 0 | Disable reverse path filtering (multi-homed RDMA) |
| `net.ipv4.conf.default.rp_filter` | 0 | Same for new interfaces |
| `vm.nr_hugepages` | 8192 | 8192 × 2MiB = 16 GiB huge pages per node |
| `vm.hugetlb_shm_group` | 0 | Allow root group to use huge pages via shm |
| `kernel.shmmax` | 68719476736 (64 GiB) | Max shared memory segment |
| `kernel.shmall` | 16777216 | Max shared memory pages |
| `kernel.perf_event_paranoid` | 1 | Allow non-root perf monitoring |

### Talos System Extensions (via Talos Factory schematic)

All 4 nodes use the same schematic including:
- `siderolabs/drbd` — DRBD kernel module
- `siderolabs/intel-ucode` — Intel CPU microcode updates
- `siderolabs/mellanox-mstflint` — Mellanox firmware flash tool
- `siderolabs/nvme-cli` — NVMe management

---

## Kubernetes RDMA Setup

### mlx5 RoCE LAG

With 802.3ad bonding active on `bond1` (both `ens2f0np0` + `ens2f1np1` as slaves), the mlx5 kernel driver automatically detects the bond and creates a single hardware LAG RDMA device: **`mlx5_bond_0`**. This has been confirmed ACTIVE on all 4 nodes.

- All RDMA traffic is load-balanced across both 25G ports in hardware (50G aggregate per node)
- `mlx5_bond_0` exposes a single `uverbs0` device to userspace via `/dev/infiniband/uverbs0`
- n02 has a broken `ens2f1np1` port — mlx5 LAG degrades gracefully to 1 active port on n02 (25G)

**Verification:**
```bash
talosctl -n 10.200.90.111 ls /sys/class/infiniband/
# Expected: mlx5_bond_0
talosctl -n 10.200.90.111 read /sys/class/infiniband/mlx5_bond_0/ports/1/state
# Expected: 4: ACTIVE
```

### RDMA Shared Device Plugin

Deployed via Flux at `flux/infrastructure/controllers/rdma-device-plugin.yaml`.

**Device selector:** `ifNames: ["ens2f0np0"]`

Even in LAG mode, `mlx5_bond_0`'s sysfs path (`/sys/class/infiniband/mlx5_bond_0/device/net/`) resolves to `ens2f0np0` (the primary bond slave). The device plugin therefore finds `mlx5_bond_0` via the `ens2f0np0` selector. Using `bond1` (virtual device, no PCIe address) would NOT work.

Exposes: `rdma/hca_shared_devices` — 1000 shared resource instances per node.

**Verification:**
```bash
kubectl get nodes -o custom-columns='NODE:.metadata.name,RDMA:.status.allocatable.rdma/hca_shared_devices'
# Expected: 1k on all 4 nodes
```

### NetworkAttachmentDefinition

Deployed via Flux at `flux/infrastructure/configs/rdma-nad.yaml`.

```yaml
name: rdma-roce
namespace: network
type: macvlan
master: bond1.292        # VLAN 292 sub-interface of the CX4 Lx bond
mode: bridge
mtu: 9000
ipam: whereabouts        # cluster-wide IP tracking (not host-local)
  range: 10.200.92.128/26
  exclude: [10.200.92.191/32]
```

`whereabouts` IPAM (deployed at `flux/infrastructure/controllers/whereabouts.yaml`) tracks allocated IPs cluster-wide via Kubernetes CRDs, preventing IP conflicts when pods on different nodes draw from the same range. `host-local` must NOT be used here — it allocates per-node from a shared range, causing duplicate IPs across nodes which breaks rdmacm peer resolution.

Pod secondary interface IPs are allocated from `10.200.92.128–10.200.92.190` (62 addresses). These are distinct from static node IPs (`10.200.92.11–14`).

**Usage in a pod:**
```yaml
metadata:
  annotations:
    k8s.v1.cni.cncf.io/networks: network/rdma-roce
spec:
  containers:
    - resources:
        limits:
          rdma/hca_shared_devices: "1"
```

---

## NIC-side QoS (PFC + ETS)

Deployed via Flux at `flux/infrastructure/configs/roce-qos-config.yaml` as DaemonSet `roce-nic-qos`.

> **Why not `tc` / `tc pedit`?** RDMA WRITE operations are submitted directly to NIC hardware via the verbs API (DMA from host memory to wire). They bypass the Linux Traffic Control (TC) subsystem entirely. A `tc pedit` rule on `bond1.292` only affects kernel-path packets (regular TCP/UDP) — it never sees NIC-generated RDMA packets.

### What the DaemonSet configures

| Step | Tool | Effect |
|------|------|--------|
| PFC reception | `dcb pfc set dev ens2f0np0 prio-pfc 3:on` | NIC pauses outgoing priority-3 frames when switch sends PFC PAUSE on priority 3 |
| ETS scheduling | `dcb ets set dev ...` | Maps prio 3→TC3 (50% ETS), prio 6→TC6 (strict), others→TC0 (50% ETS) |
| VLAN PCP | `ip link set bond1.292 type vlan egress-qos-map 3:3` | Kernel-path packets with socket priority 3 get 802.1Q PCP=3 (rdmacm CM messages etc.) |

### DSCP marking for RDMA QPs

The DSCP value in outgoing RDMA packet IP headers is set by the **application**, not the host. Applications using `libfabric` (dmf-mxl) or raw `rdmacm` must set the TOS before connecting:

```c
uint8_t tos = 0x68;  /* DSCP 26 = AF31; 26 << 2 = 0x68 */
rdma_set_option(cm_id, RDMA_OPTION_ID_TOS, &tos, sizeof(tos));
/* then: rdma_resolve_route() → rdma_connect() */
```

The mlx5 driver copies this TOS into the IP header of all RDMA packets on that QP. The switch reads `DSCP 26` and maps it to `TC3` via its `qos map ip` profile.

### Verification

```bash
# Check DaemonSet is running on all nodes
kubectl get ds roce-nic-qos -n kube-system

# Check PFC state on a node
# Run a debug pod or use kubectl exec into a privileged pod:
dcb pfc show dev ens2f0np0
# Expected: prio-pfc 3:on

dcb ets show dev ens2f0np0
# Expected: prio-tc showing 3:3 and 6:6
```

### End-to-End QoS Flow

```
Host (n01)                                Switch (sw01)
──────────────────────────────────────    ──────────────────────────────────────
App / libfabric                           trust-l3=keep
  │ rdma_set_option TOS=0x68               │ reads DSCP 26 from IP header
  │                                        │
RDMA QP (mlx5_bond_0)                     │ DSCP 26 → profile roce → TC3
  │ NIC DMA → wire (DSCP 26 in IP hdr)     │
  │                                        ├── TC3: ETS 50%, lossless (PFC)
  └─ 25G/50G LAG ─────────────────────────     ECN marking, shared pool
                                           │
PFC PAUSE ←──── switch TC3 congestion      ├── TC1/TC0: ETS 50% (storage,
  │                                        │   general traffic)
  ↓                                        │
dcb pfc prio 3 active on NIC               └── TC6: strict priority (CNP)
  NIC pauses prio-3 RDMA frames                PFC rx+tx enabled (TC3)
  until PAUSE expires                          LLDP DCBX advertising to NICs
```

---

## dmf-mxl Workload Requirements

[dmf-mxl](https://github.com/dmf-mxl/mxl) is a zero-copy shared-memory media exchange library. Its multi-node transport (PR #266, `mxl-fabrics`) uses **libfabric OFI** with `MXL_FABRICS_PROVIDER_VERBS` — libibverbs + librdmacm — to perform RDMA WRITE operations between nodes.

### What pods need

| Requirement | Kubernetes resource | Notes |
|---|---|---|
| RDMA device | `rdma/hca_shared_devices: "1"` | Exposes `mlx5_bond_0` via `/dev/infiniband/` |
| VLAN 292 IP | `k8s.v1.cni.cncf.io/networks: network/rdma-roce` annotation | macvlan on bond1.292, unique IP per pod via whereabouts |
| Shared memory (local IPC) | `emptyDir{medium: Memory}` at `/dev/shm/mxl` | tmpfs for MXL domain ring-buffer files |
| Cross-pod sharing (same node) | `hostPath: /dev/shm/mxl` instead | Allows two pods on same node to share flows directly |
| Memory lock | `securityContext.capabilities.add: [IPC_LOCK]` | Required for RDMA memory region registration |
| Libraries in image | `libibverbs`, `librdmacm`, `libfabric` (OFI verbs provider) | Build mxl with `-DMXL_ENABLE_FABRICS_OFI=ON` |

### Reference pod spec

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mxl-worker
  namespace: default
  annotations:
    # Secondary NIC on VLAN 292 — unique IP allocated by whereabouts
    k8s.v1.cni.cncf.io/networks: network/rdma-roce
spec:
  containers:
    - name: mxl-worker
      image: <your-mxl-image>  # must include libibverbs, librdmacm, libfabric
      securityContext:
        capabilities:
          add: [IPC_LOCK]
      resources:
        limits:
          rdma/hca_shared_devices: "1"
          hugepages-2Mi: "1Gi"      # optional: if using huge-page backed buffers
        requests:
          memory: "2Gi"
      volumeMounts:
        # MXL domain: tmpfs for local-only use
        - name: mxl-domain
          mountPath: /dev/shm/mxl
        # Or for cross-pod sharing on same node, use hostPath instead:
        # - name: mxl-domain
        #   mountPath: /dev/shm/mxl
        - name: dev-infiniband
          mountPath: /dev/infiniband
  volumes:
    # Local-only: each pod gets its own tmpfs MXL domain
    - name: mxl-domain
      emptyDir:
        medium: Memory
        sizeLimit: "4Gi"   # size to hold ring buffers; 1080p 10-bit v210 ≈ 6MB/frame
    # For cross-pod same-node sharing, replace with:
    # - name: mxl-domain
    #   hostPath:
    #     path: /dev/shm/mxl
    #     type: DirectoryOrCreate
    - name: dev-infiniband
      hostPath:
        path: /dev/infiniband
```

### DSCP/TOS in libfabric

The libfabric verbs provider passes rdmacm hints to the underlying connection. To set DSCP 26 (AF31) on RDMA QPs, set the `FI_OPT_CQ_DATA_SIZE` hint or use the rdmacm TOS option before connect. In mxl-fabrics the `endpointAddress.node` should be the pod's VLAN 292 IP (from the `rdma-roce` secondary interface, e.g. `10.200.92.130`).

### MXL domain URI example

```
mxl://10.200.92.130:5000/dev/shm/mxl?id=<flow-uuid>
        ↑ pod's VLAN 292 IP  ↑ libfabric listen port  ↑ MXL domain path inside pod
```

---

## Rebuild Checklist

### 1. sw01 — MikroTik CRS504-4XQ-IN

1. Flash RouterOS ≥ 7.17 (7.22.1 tested)
2. Apply the RouterOS config block from the [sw01 section](#sw01--mikrotik-crs504-4xq-in-storageroce)
3. Enable L3 HW Offloading and QoS HW Offloading: `Interfaces → Ethernet Switch → set qos-hw-offloading=yes l3-hw-offloading=yes`
4. Verify:
   ```routeros
   /interface ethernet switch qos profile print
   /interface ethernet switch qos map ip print
   /interface ethernet switch qos priority-flow-control print
   /interface ethernet switch qos port print where switch=switch1
   ```

### 2. sw02 — MikroTik CRS312-4C+8XG

1. Flash RouterOS ≥ 7.17
2. Apply the RouterOS config block from the [sw02 section](#sw02--mikrotik-crs312-4c8xg-control-plane)
3. Verify VLAN + bridge port assignments:
   ```routeros
   /interface bridge vlan print
   /interface bridge port print
   ```

### 3. Talos Nodes

```bash
cd talos/
# Generate configs
talhelper genconfig

# Apply to all nodes (no reboot required for module/sysctl changes)
for i in 1 2 3 4; do
  talosctl apply-config -n 10.200.90.11${i} -e 10.200.90.11${i} \
    -f clusterconfig/devlab-n0${i}.yaml --talosconfig clusterconfig/talosconfig
done

# Verify kernel modules
talosctl read /proc/modules -n 10.200.90.111 | grep -E "mlx5_ib|ib_uverbs|ib_umad|rdma_ucm"

# Verify RDMA char devices
talosctl ls /dev/infiniband -n 10.200.90.111
# Expected: rdma_cm, uverbs0, umad0
```

### 3. Flux / Kubernetes

```bash
# Bootstrap Flux (first time only)
flux bootstrap github \
  --token-auth \
  --owner=<github-org> \
  --repository=devlab-infra \
  --branch=main \
  --path=flux/clusters/devlab

# Force reconcile after changes
flux reconcile source git flux-system
flux reconcile kustomization infra-controllers
flux reconcile kustomization infra-configs

# Verify RDMA device plugin
kubectl logs -n kube-system -l name=rdma-shared-dp-ds --tail=5
# Expected: exposing "1000" devices on each pod

kubectl get nodes -o custom-columns='NODE:.metadata.name,RDMA:.status.allocatable.rdma/hca_shared_devices'
# Expected: 1k on all 4 nodes

# Verify whereabouts is running
kubectl get ds whereabouts -n kube-system

# Verify NIC QoS DaemonSet
kubectl get ds roce-nic-qos -n kube-system
kubectl logs -n kube-system -l app=roce-nic-qos -c configure-nic-qos --previous
# Expected: PFC prio-3 enabled, ETS configured, egress-qos-map set

# Verify mlx5 LAG is active on all nodes
for n in 111 112 113 114; do
  echo -n "n0${n: -1}: "
  talosctl -n 10.200.90.$n -e 10.200.90.$n \
    read /sys/class/infiniband/mlx5_bond_0/ports/1/state 2>&1
done
# Expected: 4: ACTIVE on all nodes
```
