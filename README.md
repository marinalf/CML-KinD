# Kind Cluster + NX-OS Fabric Lab on CML

Build notes for a `kind` (Kubernetes-in-Docker) cluster running on an Ubuntu node inside CML 2.9.

## Environment

- CML 2.9
- Ubuntu 24.04
- Nexus 9kv 

## Node settings (Settings tab)

```
RAM: 8192 MB
CPUs: 4
Boot Disk Size: 20 GB
```
(Sized for a multi-node kind cluster: control-plane + 2 workers, each running
its own kubelet/containerd overhead.)

## Initial-config

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