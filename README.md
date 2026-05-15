# Ansible Homelab

Ansible configuration for automatically setting up a Kubernetes cluster. The playbooks configure all nodes from a fresh Ubuntu Server installation to a fully working Kubernetes worker or control plane node.

## Prerequisites

### Control node (your MacBook)
- Ansible installed (`pip install ansible` in a virtualenv)
- SSH key generated (`ssh-keygen`)
- SSH key copied to all nodes (`ssh-copy-id flor@<ip>`)
- Ansible collections installed (`ansible-galaxy collection install -r requirements.yaml`)
- Vault password file created (see [Secrets](#secrets))

### Cluster nodes
- Ubuntu Server 24.04 installed
- SSH access with the `flor` user
- Static IP addresses configured via Netplan

### Important — bootstrapping
This playbook assumes that `hpprodesk01` is already configured as a Kubernetes control plane via `kubeadm init`. Worker nodes are automatically joined via `kubeadm join`, but the initial control plane setup must be done manually.

Commands for the initial control plane setup:
```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=<ip>
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## Repository structure

```
ansible-homelab/
├── ansible.cfg                  # Ansible configuration (roles path, etc.)
├── inventory.yaml               # Hosts and groups (controllers, workers, cluster)
├── requirements.yaml            # Ansible Galaxy collections
├── group_vars/
│   └── all/
│       ├── vars.yaml            # Non-secret variables
│       └── vault.yaml           # Encrypted secrets (Tailscale auth key)
├── playbooks/
│   ├── site.yaml                # Main playbook
│   └── update.yaml              # Playbook to update & upgrade all nodes
└── roles/
    ├── common/                  # Updates, swap, kernel modules, UFW
    ├── tailscale/               # Tailscale VPN installation
    ├── containerd/              # Container runtime
    ├── kubernetes/              # kubelet, kubeadm, kubectl
    ├── cilium/                  # Cilium CNI plugin
    └── cluster_addons/          # Helm, metrics-server, kube-prometheus-stack
```

## Secrets

The Tailscale auth key is stored in an Ansible Vault file. Create a `.vault_password` file in the root of the repo:

```bash
echo "your_vault_password" > .vault_password
```

This file is listed in `.gitignore` and is never committed. The vault file itself (`group_vars/all/vault.yaml`) is encrypted and safe to commit.

To edit the vault file:
```bash
ansible-vault edit group_vars/all/vault.yaml
```

## Requirements

External Ansible collections are listed in `requirements.yaml` and must be installed before running any playbook:

```bash
ansible-galaxy collection install -r requirements.yaml
```

| Collection | Used by |
|---|---|
| `artis3n.tailscale` | `tailscale` role |
| `kubernetes.core` | `cluster_addons` role |

## Usage

### Configure all nodes
```bash
ansible-playbook -i inventory.yaml playbooks/site.yaml
```

### Update & upgrade all nodes
```bash
ansible-playbook -i inventory.yaml playbooks/update.yaml
```

### Only specific nodes
```bash
ansible-playbook -i inventory.yaml playbooks/site.yaml --limit hpprodesk03,hpprodesk04
```

### Only a specific role
```bash
ansible-playbook -i inventory.yaml playbooks/site.yaml --tags kubernetes
```

### Simulate without changes (check mode)
```bash
ansible-playbook -i inventory.yaml playbooks/site.yaml --check
```

> **Note:** Check mode does not work reliably for the tailscale and join tasks as they depend on earlier tasks that are not actually executed in check mode.

## Roles

### common
Base configuration for every node:
- `apt update` and `apt upgrade`
- Disable swap (required by Kubernetes)
- Load kernel modules: `overlay` and `br_netfilter`
- Sysctl settings for Kubernetes networking
- UFW firewall rules for Kubernetes ports

### tailscale
Installs Tailscale VPN via the `artis3n.tailscale` collection. Uses the auth key from the vault.

### containerd
Installs and configures containerd as the container runtime:
- Add Docker apt repository
- Install `containerd.io`
- Set `SystemdCgroup = true` (required by Kubernetes)

### kubernetes
Installs Kubernetes packages:
- Install `kubelet`, `kubeadm`, `kubectl`
- Hold packages to prevent automatic upgrades

### cilium
Installs the Cilium CNI plugin on the control plane node:
- Downloads the Cilium CLI from the official GitHub releases
- Installs Cilium into the cluster if not already present

### cluster_addons
Installs additional cluster tooling on the control plane node:
- Install Helm
- Deploy metrics-server (with `--kubelet-insecure-tls` patch)
- Deploy kube-prometheus-stack (Prometheus + Grafana) via Helm in the `monitoring` namespace

## Inventory

```yaml
controllers:    # Control plane nodes
workers:        # Worker nodes
cluster:        # All nodes (controllers + workers)
```

## Cluster

| Hostname     | IP               | Role          |
|--------------|------------------|---------------|
| hpprodesk01  | 192.168.100.128  | control-plane |
| hpprodesk02  | 192.168.100.129  | worker        |
| hpprodesk03  | 192.168.100.130  | worker        |
| hpprodesk04  | 192.168.100.131  | worker        |
