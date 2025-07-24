
# 🧵 k8s-Cluster-on-rasp-pi5

Tutorial on how to set up a multi-node Kubernetes cluster on **Raspberry Pi 5** from scratch.

---

## 📦 Hardware Requirements

- 🖥️ 1 x Raspberry Pi 5 (Master) – 4GB minimum
- 🖥️ 1 x Raspberry Pi 5 (Worker) – 4GB minimum
- 💾 MicroSD cards or NVMe drives
- 🔌 Power supplies
- 🌐 (Optional) Network switch

---

## 📶 Network Configuration

- Assign static IPs to each Pi.
- Set hostnames:
  - Master: `k8s-master`
  - Worker: `k8s-worker`
- Enable SSH access, and update all packages.

---

## 🚀 Let's Get Started

---

### 🔧 Step 1: Update and Set Hostnames

**On Control-plane (Master):**
```bash
sudo hostnamectl set-hostname k8s-master
sudo apt update && sudo apt upgrade -y
```

**On Worker Node:**
```bash
sudo hostnamectl set-hostname k8s-worker
sudo apt update && sudo apt upgrade -y
```

**Edit `/etc/hosts` on both nodes:**
```
<MASTER_IP>   k8s-master
<WORKER_IP>   k8s-worker
```

---

### 🔧 Step 2: Disable Swap & Enable Kernel Modules

**On both nodes:**

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

**Load kernel modules:**
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

**Apply sysctl settings:**
```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

---

### 🔧 Step 3: Install `containerd` (Runtime)

**On both nodes:**
```bash
sudo apt install -y containerd

sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Use systemd as the cgroup driver
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd
```

---

### 🔧 Step 4: Install Kubernetes Components

**On both nodes:**
```bash
sudo apt install -y apt-transport-https ca-certificates curl
sudo mkdir -p /etc/apt/keyrings

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
  https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /" | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

**Verify installation:**
```bash
kubeadm version
kubelet --version
kubectl version --client
```

---

### 🧱 Step 5: Initialize Control Plane

**On `k8s-master`:**

```bash
sudo kubeadm config images pull

sudo kubeadm init \
  --apiserver-advertise-address=192.168.1.100 \
  --pod-network-cidr=192.168.0.0/16
```

⚠️ Save the `kubeadm join ...` command printed at the end.

---

### ⚙️ Step 6: Configure `kubectl` on Master

```bash
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl get nodes
```

---

### 🌐 Step 7: Install Pod Network (Calico)

**On `k8s-master`:**

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
```

---

### 🧩 Step 8: Join Worker Node

If you lost the `kubeadm join` command:
```bash
kubeadm token create --print-join-command
```

**On `k8s-worker`:**
```bash
sudo kubeadm join <MASTER_IP>:<PORT> --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

---

### ✅ Step 9: Verify the Cluster

**On `k8s-master`:**

```bash
kubectl get nodes -o wide
```

Expected output:

```
k8s-master   Ready   Control Plane
k8s-worker   Ready   Worker
```

---

### 🧪 Step 10: Deploy Test Workload

```bash
kubectl create deployment nginx --image=nginx --replicas=2
kubectl expose deployment nginx --port=80 --type=NodePort

kubectl get pods -o wide
kubectl get svc
```

Access the NGINX app via:

```
http://192.168.1.xx:<NodePort>
```

---

## 🎉 Congratulations!

Your Raspberry Pi Kubernetes cluster is now fully functional!  


---

## 📎 Resources

- [Kubernetes Official Docs](https://kubernetes.io/)
- [Project Calico](https://projectcalico.docs.tigera.io/)





ASCII Diagram
              +-----------------------------+
              |     Your Home Network       |
              |     (192.168.1.0/24)        |
              +-----------------------------+
                           |
                           |
                    +-------------+
                    | Router / WiFi|
                    +-------------+
                           |
            +-----------------------------+
            |                             |
  +-------------------+        +-------------------+
  | Raspberry Pi 4 #1 |        | Raspberry Pi 4 #2 |
  |  (Master Node)    |        |  (Worker Node)    |
  |  192.168.1.10     |        |  192.168.1.11     |
  +-------------------+        +-------------------+
            |                             |
            +-------- Kubernetes Cluster --------+
                           |  
               +-----------------------------+
               |   Deployed Nginx Service    |
               |   Exposed via NodePort or   |
               |   Nginx Reverse Proxy       |
               +-----------------------------+
