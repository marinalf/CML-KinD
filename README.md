# Kind Cluster + NX-OS Fabric Lab on CML

Build notes for a `kind` (Kubernetes-in-Docker) cluster running on an Ubuntu node inside CML 2.9.

## Environment

- CML 2.9
- Ubuntu 24.04
- 4x Nexus 9kv leaf switches (Leaf1-4), interconnected via 2 spines
- Ubuntu node has 4 dedicated fabric-facing interfaces, one per leaf:
  - `ens3` - Leaf1 `eth1/6`
  - `ens4` - Leaf2 `eth1/6`
  - `ens5` - Leaf3 `eth1/6`
  - `ens6` - Leaf4 `eth1/6`

## Node Settings

```
RAM: 8192 MB
CPUs: 4
Boot Disk Size: 20 GB
```
(Sized for a multi-node kind cluster: control-plane + 2 workers, each running its own kubelet/containerd overhead.)

## Initial Config

```yaml
#cloud-config
hostname: kind-node
ssh_pwauth: true

users:
  - name: kindops
    gecos: Kind Ops User
    groups: sudo
    sudo: ALL=(ALL) NOPASSWD:ALL
    shell: /bin/bash
    lock_passwd: false
    ssh_authorized_keys:
      - ssh-ed25519 AAAA... your-key-here

chpasswd:
  users:
    - name: kindops
      password: '<your-password-here>'
      type: text
  expire: false

write_files:
  - path: /etc/netplan/60-static.yaml
    permissions: '0600'
    content: |
      network:
        version: 2
        ethernets:
          ens2:
            dhcp4: false
            addresses: [198.18.133.5/18]
            routes:
              - to: default
                via: 198.18.128.1
            nameservers:
              addresses: [8.8.8.8]

runcmd:
  - rm -f /etc/netplan/50-cloud-init.yaml
  - netplan apply
  - curl -fsSL https://get.docker.com | sh
  - usermod -aG docker kindops
  - curl -Lo /usr/local/bin/kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
  - chmod +x /usr/local/bin/kind
  - curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
  - install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```
## Cluster Config

1 control-plane node + 3 worker nodes. The default CNI (kindnet) is disabled since Cilium will be installed separately and configured for BGP/EVPN peering with the NX-OS fabric.

`kind-config.yaml`:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  disableDefaultCNI: true
nodes:
  - role: control-plane
  - role: worker
  - role: worker
  - role: worker
```

Create the cluster on the Ubuntu node:

```
git clone https://github.com/marinalf/CML-KinD.git
cd CML-KinD
kind create cluster --name kind-cluster --config kind-config.yaml
kubectl get nodes
```

Nodes will show `NotReady` until a CNI (Cilium) is installed — expected at this stage.

## Cilium + BGP

Cilium `v1.19.5` (installed via `cilium-cli` `v0.19.6`).

Each kind node gets a second interface (`eth1`) wired straight to its own leaf switch, in addition to the existing `eth0` (`172.18.0.0/16`, cluster sync only). Cilium runs in native routing mode with the BGP Control Plane enabled, forming one eBGP session per node over its `eth1` — cluster AS 65001 to fabric AS 65000 — advertising each node's Pod CIDR to the fabric.

See [`cilium/install.md`](cilium/install.md) for the full setup (macvlan interfaces, Cilium install, BGP config apply) and [`cilium/bgp-config.yaml`](cilium/bgp-config.yaml) for the BGP Control Plane manifests.

EVPN on the NX-OS fabric side is a later, separate phase — no Cilium-side config changes are expected when it's enabled.