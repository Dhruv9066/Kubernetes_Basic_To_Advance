## 1. Launch EC2 Instances

- **Instances:** 1 control plane and 2 workers
- **Instance type:** `t2.medium`
- **AMI:** Ubuntu Server 24.04 LTS, 64-bit x86
- **Storage:** 30 GB gp3 each
- **Network:** Same VPC with private-IP connectivity and internet access

## 2. Configure Security Groups

| Destination | Protocol | Port | Source |
|---|---|---|---|
| All nodes | TCP | 22 | Your public IP |
| Control plane | TCP | 6443 | Cluster nodes and authorized administrators |
| All nodes | TCP | 10250 | Control plane |
| All nodes | UDP | 8472 | Cluster nodes |
| Application nodes | TCP/UDP | Required ports within 30000–32767 | Application clients, if using NodePort |

Allow outbound traffic for package downloads, container image pulls, and DNS resolution.

## 3. Set Hostnames

**Control plane:**

```bash
sudo hostnamectl set-hostname k8-master
```

**Worker 1:**

```bash
sudo hostnamectl set-hostname k8-worker-1
```

**Worker 2:**

```bash
sudo hostnamectl set-hostname k8-worker-2
```

## 4. Disable Swap — All Nodes

```bash
sudo swapoff -a

# Comment out any swap entries.
sudo vi /etc/fstab
```

## 5. Load Kernel Modules — All Nodes

```bash
sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

## 6. Configure Networking — All Nodes

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF

sudo sysctl --system
```

## 7. Install containerd — All Nodes

```bash
sudo apt-get update

sudo apt-get install -y \
  containerd \
  apt-transport-https \
  ca-certificates \
  curl \
  gpg

sudo mkdir -p /etc/containerd

containerd config default \
  | sudo tee /etc/containerd/config.toml > /dev/null
```

## 8. Configure containerd — All Nodes

```bash
sudo vi /etc/containerd/config.toml
```

Find `SystemdCgroup` and change its existing value to:

```toml
SystemdCgroup = true
```

Ensure `cri` is not listed in `disabled_plugins`.

```bash
sudo systemctl enable containerd
sudo systemctl restart containerd
sudo systemctl status containerd --no-pager
```

## 9. Install Kubernetes Components — All Nodes

```bash
sudo mkdir -p -m 755 /etc/apt/keyrings

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key \
  | sudo gpg --dearmor \
      -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update

sudo apt-get install -y kubelet kubeadm kubectl

sudo apt-mark hold kubelet kubeadm kubectl

sudo systemctl enable --now kubelet
```

## 10. Initialize the Cluster — Control Plane Only

Replace the example IP with your control-plane **private IP**.

```bash
CONTROL_PLANE_IP="172.31.39.80"

sudo kubeadm init \
  --kubernetes-version="$(kubeadm version -o short)" \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address="$CONTROL_PLANE_IP" \
  --cri-socket=unix:///run/containerd/containerd.sock
```

## 11. Configure kubectl — Control Plane Only

**For a regular user:**

```bash
mkdir -p "$HOME/.kube"

sudo cp -i /etc/kubernetes/admin.conf "$HOME/.kube/config"

sudo chown "$(id -u):$(id -g)" "$HOME/.kube/config"
```

**Alternatively, for root:**

```bash
export KUBECONFIG=/etc/kubernetes/admin.conf
```

## 12. Install Flannel — Control Plane Only

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

## 13. Generate the Join Command — Control Plane Only

```bash
sudo kubeadm token create --print-join-command
```

## 14. Join the Cluster — Worker Nodes Only

Run the generated command with `sudo`. Replace the placeholders below with its values.

```bash
sudo kubeadm join CONTROL_PLANE_PRIVATE_IP:6443 \
  --token GENERATED_TOKEN \
  --discovery-token-ca-cert-hash sha256:GENERATED_CA_HASH \
  --cri-socket=unix:///run/containerd/containerd.sock
```

## 15. Verify the Cluster — Control Plane Only

```bash
kubectl get nodes -o wide

kubectl get pods -A -o wide
```