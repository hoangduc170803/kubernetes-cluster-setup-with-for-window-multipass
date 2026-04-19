# Kubernetes Cluster Setup with Kubeadm on Windows

**Environment:** Multipass + Hyper-V | **K8s Version:** v1.35 | **OS:** Ubuntu 22.04 LTS

---

## Table of Contents

1. [Install Multipass on Windows](#1-install-multipass-on-windows)
2. [Create Virtual Machines with Multipass](#2-create-virtual-machines-with-multipass)
3. [Install kubeadm, kubectl, kubelet](#3-install-kubeadm-kubectl-kubelet)
4. [Install Container Runtime (containerd)](#4-install-container-runtime-containerd)
5. [Configure Static IP with Netplan](#5-configure-static-ip-with-netplan)
6. [Initialize Master Node](#6-initialize-master-node)
7. [Install CNI - Calico](#7-install-cni---calico)
8. [Join Worker Node to the Cluster](#8-join-worker-node-to-the-cluster)
9. [Verify the Cluster](#9-verify-the-cluster)

---

## 1. Install Multipass on Windows

1. Visit [multipass.run](https://multipass.run) and download the Windows installer.
2. Run the installer and follow the standard Next/Next/Finish steps.

> **Hypervisor Note:**
> - **Windows 10/11 Pro / Enterprise:** Multipass automatically uses **Hyper-V** — the native Windows hypervisor, runs smoothly.
> - **Windows Home:** Multipass uses **VirtualBox** instead (install VirtualBox first).

### Common VM Management Commands

| Command | Description |
|---------|-------------|
| `multipass list` | List VMs and their IP addresses |
| `multipass shell <name>` | SSH into a VM |
| `multipass stop <name>` | Stop a VM |
| `multipass start <name>` | Start a VM |
| `multipass delete <name>` | Delete a VM (moves to trash) |
| `multipass purge` | Permanently remove deleted VMs and reclaim disk space |

---

## 2. Create Virtual Machines with Multipass

Run the following commands on the **Windows host** (PowerShell):

```powershell
# Create Master Node
multipass launch 22.04 -n master --cpus 2 --memory 2G --disk 15G

# Create Worker Node
multipass launch 22.04 -n k8s-worker --cpus 2 --memory 2G --disk 15G
```

Wait until both commands report `Launched...`, then list the VMs and **record the IP addresses** of both machines:

```powershell
multipass list
```

Sample output (your actual IPs will differ):

```
Name          State    IPv4             Image
k8s-worker    Running  172.25.15.188    Ubuntu 22.04 LTS
master        Running  172.25.7.173     Ubuntu 22.04 LTS
```

Access the Master Node to begin setup:

```powershell
multipass shell master
```

When the prompt changes to `ubuntu@master:~$`, you are inside the VM. Type `exit` to leave.

> **Note:** From step 3 onward, all commands are run **inside the VM** unless otherwise noted.

---

## 3. Install kubeadm, kubectl, kubelet

> Run on **both Master and Worker Node**.

### 3.1 Install Dependencies

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
```

### 3.2 Add the Kubernetes APT Repository

```bash
# Create the keyring directory (if it doesn't exist)
sudo mkdir -p -m 755 /etc/apt/keyrings

# Download the Kubernetes GPG key
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Add the repository
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

### 3.3 Install and Pin the Version

```bash
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### 3.4 Enable the kubelet Service

```bash
sudo systemctl enable --now kubelet
```

---

## 4. Install Container Runtime (containerd)

> Run on **both Master and Worker Node**.

### 4.1 Enable IPv4 Packet Forwarding

```bash
# Write config (persists across reboots)
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF

# Apply immediately
sudo sysctl --system

# Verify
sysctl net.ipv4.ip_forward
# Output: net.ipv4.ip_forward = 1
```

### 4.2 Install containerd

```bash
# Download containerd
wget https://github.com/containerd/containerd/releases/download/v2.2.3/containerd-2.2.3-linux-amd64.tar.gz

# Extract to /usr/local
sudo tar -C /usr/local -xzvf containerd-2.2.3-linux-amd64.tar.gz
```

### 4.3 Configure containerd to Run with systemd

```bash
# Create the systemd service directory
sudo mkdir -p /usr/local/lib/systemd/system

# Download the service file
sudo wget -O /usr/local/lib/systemd/system/containerd.service \
  https://raw.githubusercontent.com/containerd/containerd/main/containerd.service

# Enable the service
sudo systemctl daemon-reload
sudo systemctl enable --now containerd
```

### 4.4 Install runc

```bash
# Download runc
wget https://github.com/opencontainers/runc/releases/download/v1.4.2/runc.amd64

# Install
sudo mv runc.amd64 /usr/local/sbin/runc
sudo chmod +x /usr/local/sbin/runc
```

### 4.5 Install CNI Plugins

```bash
# Download CNI plugins
sudo wget https://github.com/containernetworking/plugins/releases/download/v1.9.1/cni-plugins-linux-amd64-v1.9.1.tgz

# Extract
sudo mkdir -p /opt/cni/bin
sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.9.1.tgz
```

---

## 5. Configure Static IP with Netplan

> Run on **both Master and Worker Node** (using each node's respective IP).

### Why Static IP?

By default, Multipass uses DHCP — each time you `stop/start` a VM, the IP may change. If the Master's IP changes, the entire cluster breaks because Workers can no longer find the Master.

### 5.1 Disable Cloud-init Network Management

```bash
echo "network: {config: disabled}" \
  | sudo tee /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
```

> This prevents Cloud-init from overwriting the network config on every reboot.

### 5.2 Edit the Netplan Config File

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

Replace the entire content with the following (substitute your actual IP):

```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: no
      addresses:
        - 172.25.7.173/20       # <-- Replace with your VM's IP, subnet /20
      routes:
        - to: default
          via: 172.25.0.1       # <-- Gateway (get it with: ip route show)
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
```

### 5.3 Apply and Verify

```bash
sudo netplan apply

# Confirm the IP was assigned correctly
ip addr show eth0
# Look for: inet 172.25.7.173/20 ... valid_lft forever
```

---

## 6. Initialize Master Node

> Run only on the **Master Node**.

### 6.1 Reset Previous Configuration (if any)

```bash
sudo kubeadm reset -f
```

### 6.2 Initialize the Cluster

```bash
sudo kubeadm init \
  --pod-network-cidr=192.168.0.0/16 \
  --apiserver-advertise-address=172.25.7.173   # <-- Static IP of Master
```

> This takes about 1–2 minutes. When done, **copy the `kubeadm join` command** printed at the end — you'll need it in step 8.

### 6.3 Configure kubectl for the Current User

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 6.4 Verify the Master Node

```bash
kubectl get nodes
# Master will be in NotReady state until CNI is installed
```

---

## 7. Install CNI - Calico

> Run only on the **Master Node**.

### Why Calico with eBPF?

| Feature | iptables | eBPF (Calico) |
|---------|----------|---------------|
| Packet processing | Sequential, slow with many rules | Hash table, O(1) |
| CPU usage | Increases with rule count | Low and stable |
| Observability | Limited | Microsecond-level detail |

### 7.1 Install Tigera Operator

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.31.4/manifests/operator-crds.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.31.4/manifests/tigera-operator.yaml
```

### 7.2 Install Calico with eBPF

```bash
# Download the eBPF custom resource manifest
curl -O https://raw.githubusercontent.com/projectcalico/calico/v3.31.4/manifests/custom-resources-bpf.yaml

# Apply
kubectl create -f custom-resources-bpf.yaml
```

### 7.3 Watch the Deployment Progress

```bash
watch kubectl get tigerastatus
# Wait until all components show Available = True
```

### 7.4 Confirm Completion

```bash
kubectl get nodes
```

```
NAME     STATUS   ROLES           AGE   VERSION
master   Ready    control-plane   10m   v1.35.4
```

```bash
kubectl get pods -A
```

```
NAMESPACE         NAME                                       READY   STATUS    RESTARTS   AGE
calico-system     calico-apiserver-76f95546db-2gf8g          1/1     Running   0          3m31s
calico-system     calico-node-wh7cv                          1/1     Running   0          3m30s
kube-system       coredns-7d764666f9-7rjcl                   1/1     Running   0          10m
kube-system       etcd-master                                1/1     Running   1          10m
kube-system       kube-apiserver-master                      1/1     Running   1          10m
...
```

> **Two IP addresses on the Master explained:**
> - `172.25.7.173` — Physical IP of the Multipass VM (locked with Netplan), used for node-to-node communication.
> - `192.168.x.x` — IP of the `vxlan.calico` virtual interface created by Calico, used as the data plane "tunnel" between Pods.

---

## 8. Join Worker Node to the Cluster

> Complete steps 2, 3, and 4 on the **Worker Node** first, then join.

### 8.1 Join Using an Existing Token

Use the `kubeadm join` command printed during Master initialization. Example:

```bash
sudo kubeadm join 172.25.7.173:6443 \
  --token wexetx.oqdz90m64dt50jdc \
  --discovery-token-ca-cert-hash sha256:d09042a349ccbc7ee58fd32441b572177accd2720bcc6a7470bc3e76dc073187
```

### 8.2 Join When the Token Has Expired (> 24h)

The default token expires after **24 hours**. Run on the **Master Node** to generate a new one:

```bash
# Create a new token and print the join command
sudo kubeadm token create --print-join-command
```

If you need to retrieve each part manually:

```bash
# List valid tokens
sudo kubeadm token list

# Get the discovery token CA cert hash
sudo cat /etc/kubernetes/pki/ca.crt \
  | openssl x509 -pubkey \
  | openssl rsa -pubin -outform der 2>/dev/null \
  | openssl dgst -sha256 -hex \
  | sed 's/^.* //'
```

---

## 9. Verify the Cluster

Run on the **Master Node** after the Worker has joined:

```bash
kubectl get nodes
```

Expected output (both nodes `Ready`):

```
NAME         STATUS   ROLES           AGE   VERSION
master       Ready    control-plane   15m   v1.35.4
k8s-worker   Ready    <none>          2m    v1.35.4
```

---

## Setup Flow Summary

```
[Windows Host]
    ├─ Step 1: Install Multipass (multipass.run) → Hyper-V (Pro) or VirtualBox (Home)
    ├─ Step 2: multipass launch ubuntu -n master      → Create Master Node
    └─ Step 2: multipass launch ubuntu -n k8s-worker  → Create Worker Node

[BOTH nodes — Steps 3, 4, 5]
    ├─ Install kubeadm, kubectl, kubelet
    ├─ Install containerd + runc + CNI plugins
    └─ Configure static IP (Netplan + disable Cloud-init)

[MASTER only — Steps 6, 7]
    ├─ kubeadm init --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address=<IP>
    ├─ Configure kubectl ($HOME/.kube/config)
    └─ Install Calico CNI (Tigera Operator + eBPF)

[WORKER only — Step 8]
    └─ kubeadm join <master-ip>:6443 --token ... --discovery-token-ca-cert-hash ...

[Verify on MASTER — Step 9]
    └─ kubectl get nodes  →  both nodes STATUS = Ready ✓
```
