# K3s Kubernetes Single-Node Install with Ansible

A tiny Ansible playbook to install **K3s (lightweight Kubernetes)** on a Linux server via SSH.  
Designed to be beginner-friendly and easy to copy/paste.

> **Why Ansible if K3s already has an installer?**
> You can absolutely perform a quick, hands-on install of K3s using the official script at https://get.k3s.io. This project leverages that same installer but wraps it with Ansible so you can apply the exact, repeatable steps **at scale**—ideal for corporate environments where a **single‑node** Kubernetes makes sense. Typical use cases include lab environments, learning/testing, CI/automation runners, edge/IoT gateways, or any containerized workloads that don’t require the high availability of a full multi‑node cluster.

> Repo name suggestion: `kubernetes-light-k3s-install-ansible`  
> Playbook entrypoint: `playbook-k3s-install.yml`

---

## Contents

- [What this is](#what-this-is)
- [Prerequisites](#prerequisites)
- [Install Ansible (control machine)](#install-ansible-control-machine)
  - [Option A: Ubuntu/Debian via APT](#option-a-ubuntudebian-via-apt)
  - [Option B: Any distro via pipx](#option-b-any-distro-via-pipx)
- [Prepare SSH access](#prepare-ssh-access)
- [Create an inventory](#create-an-inventory)
- [Optional: ansible.cfg for nicer defaults](#optional-ansiblecfg-for-nicer-defaults)
- [How the playbook works](#how-the-playbook-works)
- [Run the playbook](#run-the-playbook)
- [After install: use your cluster](#after-install-use-your-cluster)
- [Uninstall / reset](#uninstall--reset)
- [Troubleshooting](#troubleshooting)
- [FAQ](#faq)
- [License](#license)

---

## What this is

This repository automates the **installation of a single-node K3s** on a target Linux server:

- Installs K3s using the official installer (`get.k3s.io`)
- Enables and starts the `k3s` systemd service
- Leaves a kubeconfig at `/etc/rancher/k3s/k3s.yaml`
- Installs `kubectl` on the node (bundled by K3s)
- Install `helm` on the node to manage the installation of new kubernetes packages
  
---

## Prerequisites

**Control machine (your laptop/desktop):**
- Linux (Ubuntu/Debian/Fedora/Arch/etc.)
- SSH access to the target host (key-based recommended)
- Ansible installed

**Target server (where K3s will run):**
- A fresh Linux VM/host with a user that can `sudo`
- SSH open from the control machine
- Internet access to reach `https://get.k3s.io` (for online installs)

---

## Install Ansible (control machine)

### Option A: Ubuntu/Debian via APT

```bash
sudo apt update
sudo apt install -y software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install -y ansible
ansible --version
```

### Option B: Any distro via pipx

```bash
# On Ubuntu/Debian (install pipx once):
sudo apt update && sudo apt install -y pipx
pipx ensurepath

# Install the Ansible CLI (isolated, no root needed):
pipx install --include-deps ansible
ansible --version
```

---

## Prepare SSH access

Generate a key (if you don’t have one) and copy it to the server:

```bash
ssh-keygen -t ed25519 -C "you@example.com"
ssh-copy-id <your-user>@<server-ip>

# Quick connectivity check:
ssh <your-user>@<server-ip> "hostnamectl --static"
```

---

## Create an inventory

Create a file named **`hosts.ini`** in the repo root:

```ini
[k3s_server]
my-k3s ansible_host=192.168.56.10 ansible_user=ubuntu ansible_ssh_common_args='-o StrictHostKeyChecking=no'

[k3s_server:vars]
ansible_become=true
```

Validate connectivity:

```bash
ansible-inventory -i hosts.ini --list
ansible k3s_server -i hosts.ini -m ping
```

> Tip: if your sudo is passwordless on the target, you can omit `--ask-become-pass` later.

---

## Optional: ansible.cfg for nicer defaults

Create a local **`ansible.cfg`** in the repo root to make output faster and cleaner:

```ini
[defaults]
inventory = ./hosts.ini
host_key_checking = False
deprecation_warnings = False
stdout_callback = yaml
bin_ansible_callbacks = True
interpreter_python = auto_silent

[ssh_connection]
pipelining = True
```

This lets you run `ansible-playbook playbook-k3s-install.yml` without passing `-i hosts.ini` every time.

---

## How the playbook works

`playbook-k3s-install.yml` is a single play that targets the `k3s_server` group. At a high level it:

1. Ensures basic packages/tools are present (e.g., `curl`).
2. Runs the **official K3s installer** from `get.k3s.io`.
3. Enables/starts the `k3s` systemd service.
4. Leaves you with:
   - `kubectl` available on the node,
   - kubeconfig at `/etc/rancher/k3s/k3s.yaml`,
   - helper script `k3s-uninstall.sh`.

> Customization (optional): manage `/etc/rancher/k3s/config.yaml` via a template to set flags like `disable: ["traefik"]`, `tls-san`, `write-kubeconfig-mode`, etc., **before** starting K3s.

Example `config.yaml` you can template later:

```yaml
# /etc/rancher/k3s/config.yaml
write-kubeconfig-mode: "0644"
disable:
  - traefik
tls-san:
  - my-k3s.local
  - 192.168.56.10
kube-apiserver-arg:
  - feature-gates=EphemeralContainers=true
```

---

## Run the playbook

From the repository root:

```bash
ansible-playbook playbook-k3s-install.yml --ask-become-pass
```

Useful flags:

- Dry run first: `--check`
- Show config changes: `--diff`
- Limit to one host: `-l my-k3s`
- Increase verbosity: `-v` / `-vv` / `-vvv`

---

## After install: use your cluster

On the **server**, check the kubeconfig and node status:

```bash
sudo cat /etc/rancher/k3s/k3s.yaml | head -n 5
kubectl get nodes -o wide
```

If you want to manage this k3s kubernetes from your **laptop**:

```bash
# Copy kubeconfig to your machine:
scp <your-user>@<server-ip>:/etc/rancher/k3s/k3s.yaml ~/.kube/config

# (Optional) Edit the 'server:' address in ~/.kube/config to the node's reachable IP/DNS.
kubectl get nodes
kubectl get pods -A
```

---

## Uninstall / reset

**Warning:** This is destructive; it removes K3s and local cluster data from the node.

```bash
sudo /usr/local/bin/k3s-uninstall.sh
```

This stops K3s and removes files created by the installer (PVCs on external storage are not affected).

---

## Troubleshooting

- **Permission denied reading kubeconfig**  
  Read with sudo on the node:
  ```bash
  sudo cat /etc/rancher/k3s/k3s.yaml
  ```
  Or set `write-kubeconfig-mode: "0644"` in `/etc/rancher/k3s/config.yaml` (only if appropriate for your security model).

- **Service not running**  
  Check systemd and logs:
  ```bash
  sudo systemctl status k3s
  sudo journalctl -u k3s -f
  ```

- **Air-gapped installs**  
  Download the installer and images to an internal mirror; adjust the playbook to use local artifacts.

- **Firewall / ports**  
  Ensure the node can reach the Internet during install, and that you can reach the API server port (usually `6443`) if accessing remotely.

---

## License

This project is licensed under the terms of the repository’s `LICENSE` file.
