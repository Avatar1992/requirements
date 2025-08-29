Great — below is a **clear, numbered, step-by-step guide** to set up a kubeadm Kubernetes cluster on AWS EC2 **(one control-plane / master + one or more workers)** — **no shell scripts**, just one command / one manual edit at a time and explanation for each step. Run every command as `sudo` (or as root) unless stated otherwise.

---

## ⚠️ Prerequisites (before you start)

1. **EC2 instances**

   * 1 master (control-plane): **Ubuntu 22.04 or 24.04**, 2 vCPU / 4 GB RAM (t2.medium / t3.medium or similar)
   * 1+ worker(s): Ubuntu, 2 GB+ RAM
2. **Security group inbound rules** (allow between nodes):

   * SSH: `22` (your IP)
   * Kubernetes API: `6443` (master)
   * etcd (if HA control plane): `2379-2380` (master ↔ master) — only needed for multi-control plane
   * Kubelet: `10250` (master ↔ nodes)
   * kube-scheduler / kube-controller manager: `10251`, `10252`
   * NodePort range (if needed externally): `30000-32767`
3. **Private networking**: place nodes in same VPC/subnet or ensure routeability (use private IPs).
4. **SSH access**: you must be able to SSH to each node as `ubuntu` (or similar) user.

---

# Step-by-step (manual, no scripts)

### 1) Set hostnames and update OS

On **each node** (master and workers) do:

* Set a distinct hostname:

```bash
sudo hostnamectl set-hostname master-node      # on master
sudo hostnamectl set-hostname worker-1         # on each worker
```

* Update packages:

```bash
sudo apt update && sudo apt upgrade -y
```

**Why:** clean system state and readable node names in `kubectl get nodes`.

---

### 2) Disable swap (Kubernetes requires this)

On **each node**:

* Turn off swap immediately:

```bash
sudo swapoff -a
```

* Make it persistent: open `/etc/fstab` in editor and comment out any swap line (put `#` at start). For example:

```
# /swapfile none swap sw 0 0
```

**Why:** kubelet refuses to run with swap enabled.

---

### 3) Configure kernel networking (required for pod networking)

On **each node**:

* Load modules:

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

* Create sysctl config file: open `/etc/sysctl.d/k8s.conf` in an editor and paste:

```
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
```

Then apply:

```bash
sudo sysctl --system
```

**Why:** lets bridged network traffic be seen by iptables, required for CNIs (Calico, Flannel, etc.).

---

### 4) Install and configure containerd (recommended runtime)

On **each node**:

* Install prerequisites and containerd:

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release
sudo apt install -y containerd
```

* Generate default containerd config file:

```bash
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
```

* Edit `/etc/containerd/config.toml` with an editor (e.g., `sudo nano /etc/containerd/config.toml`) and locate the line:

```
# SystemdCgroup = false
```

Change it to:

```
SystemdCgroup = true
```

(If the exact line differs, find `SystemdCgroup` under `plugins."io.containerd.grpc.v1.cri".containerd.runtimes.*.options` and set to `true`.)

* Restart containerd:

```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
```

* Verify containerd is running:

```bash
containerd --version
sudo systemctl status containerd
```

**Why:** kubelet defaults to `systemd` cgroup driver on modern distros; containerd must match that for stable operation.

---

### 5) Add Kubernetes apt repository and install kubeadm, kubelet, kubectl

On **each node**:

* Create apt keyrings directory:

```bash
sudo mkdir -p /etc/apt/keyrings
```

* Download the Kubernetes release key and add repository (example uses v1.30 stable repository; adjust version if needed):

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

* Update and install:

```bash
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
```

* Prevent automatic upgrades (hold packages):

```bash
sudo apt-mark hold kubelet kubeadm kubectl
```

* Verify:

```bash
kubeadm version
kubelet --version
kubectl version --client
```

**Why:** adds official Kubernetes packages; holding avoids unexpected breaking upgrades.

---

### 6) Initialize the control-plane (master) — only on the master node

On **the master node only**:

* Choose a pod CIDR compatible with the CNI you will use. For Calico use `192.168.0.0/16`. Example init:

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

* The `kubeadm init` output shows important info:

  * `kubeadm join ...` command for worker nodes (copy it and keep it safe)
  * `kubeadm token create --print-join-command` can also be used to regenerate the join command

* To manage the cluster with `kubectl` as the current (non-root) user, set up kubeconfig:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

(If you ran `sudo` and want to use the ubuntu user, adjust `$HOME` or copy to `/home/ubuntu/.kube/config` and chown to ubuntu.)

**Why:** initializes control plane components (`kube-apiserver`, `etcd`, `controller-manager`, `scheduler`) and generates the join token.

---

### 7) Install a Pod network add-on (CNI) — on the master

Choose a CNI and apply it (Calico example):

* Apply Calico:

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.3/manifests/calico.yaml
```

* Verify CNI pods:

```bash
kubectl get pods -n kube-system
```

Wait until all relevant pods are `Running`.

**Why:** Pod networking must be configured so pods on different nodes can reach each other. `--pod-network-cidr` used in `kubeadm init` must match CNI expectations.

---

### 8) Join worker nodes to the cluster

On **each worker node**:

* Run the `kubeadm join` command you copied from the master (it looks like):

```bash
sudo kubeadm join <MASTER_PRIVATE_IP>:6443 --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
```

* After joining, back on the master verify:

```bash
kubectl get nodes
```

You should see the worker(s) with `STATUS` **Ready** after some time.

**Why:** worker runs kubelet and container runtime; join registers the node with the control plane.

---

### 9) Post-installation checks & helpful commands

On the master:

* Check nodes and conditions:

```bash
kubectl get nodes
kubectl describe node <node-name>
```

* Check all system pods:

```bash
kubectl get pods -A
```

* If pods are pending, check events:

```bash
kubectl get events -A --sort-by=.metadata.creationTimestamp
```

* See kubelet logs for troubleshooting:

```bash
sudo journalctl -u kubelet -f
```

**Why:** verifies cluster health and helps diagnose issues.

---

### 10) (Optional) Allow scheduling on master (for single-node dev)

If you want to run pods on the master (single-node or testing):

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane- || true
```

**Why:** by default control-plane nodes are tainted so user pods don’t schedule there.

---

## Common problems & how to fix them

* **`kubeadm/kubelet/kubectl` unavailable**: you didn’t add the Kubernetes apt repo correctly — repeat Step 5.
* **Node stays `NotReady`**: check `kubectl get pods -n kube-system` and `journalctl -u kubelet`. Often caused by:

  * Missing CNI (apply Calico/Flannel)
  * Swap not disabled
  * cgroup driver mismatch (containerd must use `SystemdCgroup = true`)
* **`cgroup driver mismatch` error**: ensure containerd has `SystemdCgroup = true` and kubelet uses systemd (kubeadm defaults to systemd).
* **Firewall blocking ports**: verify Security Group and instance `ufw`/iptables settings.
* **Pods stuck in `ContainerCreating`**: check `kubectl describe pod` and `kubectl logs` for the pod; likely image pull or CNI issue.

---

## Quick verification checklist (after all steps)

* From master:

  * `kubectl get nodes` → master + workers show `Ready`
  * `kubectl get pods -A` → core system pods `Running`
  * `kubectl get svc -n kube-system` → CNI services exist

---

## Notes & tips

* Use **private IPs** between nodes for `kubeadm join`. In AWS, use the instance private IP (not public).
* Keep `kubeadm`, `kubelet`, `kubectl` versions consistent across nodes.
* For **production**, consider HA control plane, external etcd, proper TLS and backup strategies.
* If you want, I can provide the exact **one-by-one commands to copy into each node’s terminal** (master vs worker) so you can follow interactively.

---

Would you like a concise **printable checklist** (one page PDF) or a version that lists **one terminal command per line** for master and worker, so you can copy-paste interactively?
