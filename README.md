# Kubernetes Home Lab Automation

Automated step-by-step deployment of a Kubernetes v1.36.1 cluster using Ansible, `kubeadm`, and KVM/QEMU virtual machines.


## Architecture Summary

* **Control Plane Node**: Ubuntu (Localhost)
* **Worker Node 1 (`node-1`)**: Rocky Linux / RHEL 
* **Worker Node 2 (`node-2`)**: Rocky Linux / RHEL 
* **Pod Network CIDR**: `10.244.0.0/16`
* **Service CIDR**: `10.96.0.0/12`


## Step 1: Install KVM/QEMU on Host Machine

Run the following commands on your host system (Ubuntu) to install KVM, QEMU, and necessary management utilities:

```bash
# Update system package index
sudo apt update

# Install KVM, QEMU, libvirt, and network tools
sudo apt install qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils virtinst virt-manager -y

# Enable and start the virtualization daemon
sudo systemctl enable --now libvirtd

# Add current user to libvirt and kvm groups
sudo usermod -aG libvirt $(whoami)
sudo usermod -aG kvm $(whoami)

```


## Step 2: Install Control & Target Dependencies

### 2.1 Control Node (Ubuntu Localhost)

Install Ansible and Python package manager:

```bash
sudo apt update
sudo apt install ansible python3-pip -y

```

### 2.2 Target Nodes (Rocky Linux / RHEL)

Ansible requires Python 3 and SELinux bindings to run modules on RHEL-based targets. Execute this on `node-1` and `node-2`:

```bash
sudo dnf install python3 python3-libselinux -y

```

---

## Step 3: Configure Passwordless SSH & Sudo Escalation

### 3.1 Generate & Distribute SSH Keys

Generate an SSH key pair on your control node and copy it to both worker nodes for user `u`:

```bash
# Generate SSH Key (Press enter to accept default paths)
ssh-keygen -t rsa -b 4096 -N ""

# Copy SSH public key to worker nodes
ssh-copy-id <username>@<ip_adress>

```

### 3.2 Configure Passwordless Sudo

Configure `sudoers` privileges on the target worker nodes so Ansible can escalate commands (`become: true`) without password prompts:

```bash
ssh u@<ip_address> "echo 'u ALL=(ALL) NOPASSWD: ALL' | sudo tee /etc/sudoers.d/u"
ssh u@<ip_address> "echo 'u ALL=(ALL) NOPASSWD: ALL' | sudo tee /etc/sudoers.d/u"

```

---

## Step 4: Verify Inventory Connectivity

Ensure your project contains the `ansible.cfg` and `hosts` files configured properly:

```bash
# Test Ansible connectivity across master and worker nodes
ansible all -m ping -i hosts

```

---

## Step 5: Execute Cluster Provisioning Playbooks

Run the playbooks in exact sequential order to bootstrap the Kubernetes cluster:

### Step 5.1: Apply System Prerequisites

Disables swap, configures required kernel modules (`overlay`, `br_netfilter`), and sets `sysctl` net-bridge parameters:

```bash
ansible-playbook k8s-prerequisites.yml

```

### Step 5.2: Install Container Runtime

Installs `containerd` across all nodes and enables `SystemdCgroup` driver:

```bash
ansible-playbook containerd.yml

```

### Step 5.3: Install Kubernetes Packages

Installs `kubelet`, `kubeadm`, and `kubectl` on the master and worker nodes:

```bash
ansible-playbook install-k8s-ubuntu.yml
ansible-playbook install-k8s-rhel.yml

```

### Step 5.4: Initialize Control Plane

Bootstraps the master node, sets up `/root/.kube/config`, and captures the join command:

```bash
ansible-playbook init-control-plane.yml

```

### Step 5.5: Join Worker Nodes

Fetches token dynamically from control plane and joins worker nodes (`node-1`, `node-2`) to cluster:

```bash
ansible-playbook join-workers.yml

```

---

## Step 6: Verify Cluster Status

Log into your master node or run as root to verify that all nodes are in `Ready` state:

```bash
kubectl get nodes -o wide

```

---

## Step 7: Cluster Maintenance & Upgrades

To upgrade cluster versions safely (one node at a time with drain/uncordon handling):

* **Upgrade Control Plane (Ubuntu):**
```bash
ansible-playbook k8s-version-upgrades-ubuntu.yml -e "target_k8s_version=1.36.1 target_k8s_package_version=1.36.1-1.1"

```


* **Upgrade Worker Nodes (Rocky Linux):**
```bash
ansible-playbook k8s-version-upgrades-rocky.yml -e "target_k8s_package_version=1.36.1"

```

