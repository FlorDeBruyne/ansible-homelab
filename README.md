# Ansible Homelab

Ansible configuration for automatically setting up a Kubernetes cluster. The playbooks configure all nodes from a fresh Ubuntu Server installation to a fully working Kubernetes worker or control plane node.

## Prerequisites

### Control node
- Ansible installed (`pip install ansible` in a virtualenv)
- SSH key generated (`ssh-keygen`)
- SSH key copied to all nodes (`ssh-copy-id $USER@<ip>`)
- Ansible collections installed (`ansible-galaxy collection install -r requirements.yaml`)
- Vault password file created (see [Secrets](#secrets))

### Cluster nodes
- Ubuntu Server 24.04 installed
- SSH access with the `$USER` user
- Static IP addresses configured via Netplan

## Repository structure

```
ansible-homelab/
├── ansible.cfg                  # Ansible configuration (roles path, etc.)
├── inventory.yaml               # Hosts and groups (controllers, workers, cluster)
├── requirements.yaml            # Ansible Galaxy collections
├── group_vars/
│   └── all/
│       ├── vars.yaml            # Non-secret variables (kubeconfig path, pod CIDR)
│       └── vault.yaml           # Encrypted secrets (Tailscale auth key)
├── playbooks/
│   ├── site.yaml                # Main playbook
│   ├── update.yaml              # Playbook to update & upgrade all nodes
│   └── update_kubernetes.yaml   # Playbook to upgrade Kubernetes version
└── roles/
    ├── common/                  # Updates, swap, kernel modules, UFW, kubelet IP fix
    ├── tailscale/               # Tailscale VPN installation
    ├── containerd/              # Container runtime
    ├── kubernetes/              # kubelet, kubeadm, kubectl
    ├── controller_init/         # kubeadm init, kubeconfig, Helm, Cilium, Flux CLI
    └── storage/                 # 1TB disk partitioning and mounting for Longhorn
```

## Secrets

The following secrets are stored in an Ansible Vault file:
- Tailscale auth key

Create a `.vault_password` file in the root of the repo:

```bash
echo "your_vault_password" > .vault_password
```

This file is listed in `.gitignore` and is never committed. The vault file itself (`group_vars/all/vault.yaml`) is encrypted and safe to commit.

To edit the vault file:
```bash
ansible-vault edit group_vars/all/vault.yaml
```

The vault file should contain:
```yaml
vault_tailscale_authkey: "tskey-auth-xxxxx"
```

## Requirements

External Ansible collections are listed in `requirements.yaml` and must be installed before running any playbook:

```bash
ansible-galaxy collection install -r requirements.yaml
```

| Collection | Used by |
|---|---|
| `artis3n.tailscale` | `tailscale` role |
| `kubernetes.core` | `controller_init` role |
| `community.general` | `storage` role |
| `ansible.posix` | `storage` role |

## Usage

### Configure all nodes
```bash
ansible-playbook -i inventory.yaml playbooks/site.yaml
```

### Update & upgrade all nodes
```bash
ansible-playbook -i inventory.yaml playbooks/update.yaml
```

### Upgrade Kubernetes to a specific version
```bash
ansible-playbook -i inventory.yaml playbooks/update_kubernetes.yaml -e "k8s_version=1.33.0-1.1"
```

> **Note:** If no version is specified, the playbook defaults to `1.32.0-1.1`.

### Only specific nodes
```bash
ansible-playbook -i inventory.yaml playbooks/site.yaml --limit hpprodesk03,hpprodesk04
```

### Only a specific role
```bash
ansible-playbook -i inventory.yaml playbooks/site.yaml --tags controller_init
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
- Sudo access for the `$USER` user
- Disable swap (required by Kubernetes)
- Load kernel modules: `overlay` and `br_netfilter`
- Sysctl settings for Kubernetes networking
- UFW firewall rules for Kubernetes ports
- Configure kubelet to use the LAN IP instead of the Tailscale IP

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
- Enable kubelet service

### controller_init
Full control plane setup — only runs on the controller node:
- Run `kubeadm init` (idempotent — skipped if already initialized)
- Set up kubeconfig for the `$USER` user
- Install Helm
- Install Cilium CNI plugin
- Install Flux CLI (for use with FluxCD bootstrap in the Kubernetes repo)

### storage
Prepares the 1TB disk (`/dev/sda`) on all nodes for use with Longhorn:
- Removes existing partitions (idempotent — skipped if already mounted)
- Creates a new primary ext4 partition
- Mounts the partition at `/mnt/longhorn`

## Kubernetes upgrade

The `update_kubernetes.yaml` playbook upgrades Kubernetes on all nodes in the correct order:

1. Upgrade the controller node
2. Drain each worker node one by one (`serial: 1`)
3. Upgrade each worker node
4. Uncordon each worker node after upgrade

Run with a specific version:
```bash
ansible-playbook -i inventory.yaml playbooks/update_kubernetes.yaml -e "k8s_version=1.33.0-1.1"
```

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