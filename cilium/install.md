# Cilium install + fabric-facing interfaces

Run all of this on the CML Ubuntu node.

## 0. Bring up the fabric-facing interfaces

`ens3`-`ens6` come up administratively **down** by default. Docker's macvlan
needs the parent interface up to pass any traffic. Persist this via netplan
(same mechanism already managing `ens2`) so it survives a reboot — these
interfaces don't need an IP of their own, just link-up, since Docker's
macvlan handles addressing on the container side:

```
sudo tee /etc/netplan/61-fabric-interfaces.yaml > /dev/null << 'EOF'
network:
  version: 2
  ethernets:
    ens3:
      dhcp4: false
      optional: true
    ens4:
      dhcp4: false
      optional: true
    ens5:
      dhcp4: false
      optional: true
    ens6:
      dhcp4: false
      optional: true
EOF
sudo netplan apply
```

Confirm:

```
ip -br link show | grep -E 'ens3|ens4|ens5|ens6'
```

All four should show `UP` and `LOWER_UP`.

## 1. Give each kind node a dedicated fabric interface

Each kind node keeps `eth0` (`172.18.0.0/16`, kind's default Docker bridge) for cluster-internal traffic only, and gets a second interface `eth1` wired straight to its own leaf switch via a Docker macvlan network on top of the host's physical uplink.

Mapping:

| Node                         | Host interface | Leaf  | Link subnet     | Leaf IP      | Node IP      |
|-------------------------------|-----------------|-------|------------------|--------------|--------------|
| `kind-cluster-control-plane`  | `ens3`          | Leaf1 | `198.19.0.0/30`  | `198.19.0.1` | `198.19.0.2` |
| `kind-cluster-worker`         | `ens4`          | Leaf2 | `198.19.0.4/30`  | `198.19.0.5` | `198.19.0.6` |
| `kind-cluster-worker2`        | `ens5`          | Leaf3 | `198.19.0.8/30`  | `198.19.0.9` | `198.19.0.10`|
| `kind-cluster-worker3`        | `ens6`          | Leaf4 | `198.19.0.12/30` | `198.19.0.13`| `198.19.0.14`|

Create one macvlan network per host interface:

```
docker network create -d macvlan --subnet=198.19.0.0/30 -o parent=ens3 leaf1-net
docker network create -d macvlan --subnet=198.19.0.4/30 -o parent=ens4 leaf2-net
docker network create -d macvlan --subnet=198.19.0.8/30 -o parent=ens5 leaf3-net
docker network create -d macvlan --subnet=198.19.0.12/30 -o parent=ens6 leaf4-net
```

Attach each network to its mapped node with a fixed IP:

```
docker network connect --ip=198.19.0.2  leaf1-net kind-cluster-control-plane
docker network connect --ip=198.19.0.6  leaf2-net kind-cluster-worker
docker network connect --ip=198.19.0.10 leaf3-net kind-cluster-worker2
docker network connect --ip=198.19.0.14 leaf4-net kind-cluster-worker3
```

Verify `eth1` came up on each node:

```


docker exec kind-cluster-control-plane ip -4 addr show eth1
docker exec kind-cluster-worker        ip -4 addr show eth1
docker exec kind-cluster-worker2       ip -4 addr show eth1
docker exec kind-cluster-worker3       ip -4 addr show eth1
```

## 2. Install the Cilium CLI

```
curl -L --fail --remote-name-all \
  https://github.com/cilium/cilium-cli/releases/latest/download/cilium-linux-amd64.tar.gz
sudo tar xzvfC cilium-linux-amd64.tar.gz /usr/local/bin
rm cilium-linux-amd64.tar.gz
```

## 3. Install Cilium

Native routing (not the default VXLAN encap) so Pod CIDR routes are real, BGP-advertised routes to the fabric, plus the BGP Control Plane feature:

```
cilium install \
  --set routingMode=native \
  --set ipv4NativeRoutingCIDR=10.244.0.0/16 \
  --set bgpControlPlane.enabled=true
```

Wait for it to come up:

```
cilium status --wait
kubectl get nodes
```

Nodes should flip from `NotReady` to `Ready`.

## 4. Apply the BGP config

```
kubectl apply -f bgp-config.yaml
```

## 5. Verify

```
cilium bgp peers
cilium bgp routes
```

All 4 sessions should show `established` (remote AS 65000). Cross-node pod reachability (e.g. a pod on `control-plane` reaching a pod on `worker3`, which sits behind a different leaf) confirms the spine is carrying traffic between leaves for this underlay.
