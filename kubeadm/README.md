---
created: 2024-07-27T00:47:23+05:30
updated: 2024-07-27T13:18:06+05:30
Maintainer: Ibrar Ansari
---
# Kubeadm Installation Guide

This guide outlines the steps needed to set up a Kubernetes cluster using kubeadm.
## My Cluster info

|Name|IP|CPU|Memory|Storage|
|---|---|---|---|---|
|Master|192.168.1.205|2|4|30|
|Node1|192.168.1.201|2|4|25|
|Node2|192.168.1.202|2|4|25|
|Node3|192.168.1.203|2|4|25|
|Node3|192.168.1.204|2|4|25|

## Pre-requisites

- Ubuntu OS (Xenial or later)
- sudo privileges
- Internet access
- 2Core, 4GB or higher

## Firewall Setup

> Required ports for Control Plane
```
sudo ufw allow 6443/tcp
sudo ufw allow 2379:2380/tcp
sudo ufw allow 10250/tcp
sudo ufw allow 10259/tcp
sudo ufw allow 10257/tcp
```
> Required ports for Worker Nodes
```
sudo ufw allow 10250/tcp #Kubelet API
sudo ufw allow 30000:32767/tcp #NodePort Services
```
>  Required ports for Calico CNI
```
sudo ufw allow 179/tcp
sudo ufw allow 4789/udp
sudo ufw allow 4789/tcp
sudo ufw allow 2379/tcp
```

---
## Execute on Both "Master" & "Worker Node"

Run the following commands on both the master and worker nodes to prepare them for kubeadm.

```bash
# Install basic app
sudo apt-get update -y
sudo apt install -y iputils-ping net-tools nano vim telnet jq curl gnupg2 gpg software-properties-common apt-transport-https ca-certificates

# First, disable the swap to make kubelet work properly
sudo sed -i '/\bswap\b/s/^/#/' /etc/fstab
sudo swapoff -a

# Create the .conf file to load the modules at bootup
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

# Configure required modules
sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

cat /etc/sysctl.d/k8s.conf

# Enable IP Forwarding
sudo sh -c "echo 'net.ipv4.ip_forward = 1' >> /etc/sysctl.conf"
sudo sysctl -p
cat /proc/sys/net/ipv4/ip_forward

# Apply sysctl params without reboot
sudo sysctl --system

## Install CRIO Runtime
sudo curl -fsSL https://pkgs.k8s.io/addons:/cri-o:/prerelease:/main/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://pkgs.k8s.io/addons:/cri-o:/prerelease:/main/deb/ /" | sudo tee /etc/apt/sources.list.d/cri-o.list
sudo apt-get update -y
sudo apt-get install -y cri-o
sudo systemctl daemon-reload
sudo systemctl enable crio --now
sudo systemctl start crio.service
sudo systemctl status crio.service

# Add Kubernetes APT repository and install required packages
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update -y
sudo apt-get install -y kubelet="1.29.0-*" kubectl="1.29.0-*" kubeadm="1.29.0-*"
sudo apt-mark hold kubeadm kubelet kubectl
sudo systemctl enable --now kubelet
sudo systemctl start kubelet
sudo systemctl status kubelet
kubeadm version
```

---

## Execute ONLY on "Master Node"

```bash
# Pull Image
sudo kubeadm config images pull
sudo kubeadm config images list

# Launch Kubernetes
sudo kubeadm init

# Load profile
export KUBECONFIG=/etc/kubernetes/admin.conf

mkdir -p "$HOME"/.kube
sudo cp -i /etc/kubernetes/admin.conf "$HOME"/.kube/config
sudo chown "$(id -u)":"$(id -g)" "$HOME"/.kube/config

# Network Plugin = calico
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.0/manifests/calico.yaml

kubeadm token create --print-join-command

- You will get `kubeadm token`, **Copy it**.
```

- You will get `kubeadm token`, **Copy it**.

---

## Execute on ALL of your Worker Node's

1. Perform pre-flight checks

   ```bash
   sudo kubeadm reset pre-flight checks
   ```

2. Paste the join command you got from the master node and append `--v=5` at the end.

   ```bash
   sudo your-token --v=5
   ```

   > Use `sudo` before the token.

---

## Verify Kubernetes Cluster

**On Master Node:**

```bash
kubectl cluster-info
kubectl get nodes
kubectl get nodes -o wide
```

---

## Optional: Labeling Nodes

If you want to label worker nodes, you can use the following command:

```bash
kubectl label node <node-name> node-role.kubernetes.io/worker=worker
```

---

## Optional: Test a Nginx Pod

If you want to test a Nginx pod, you can use the following command:

```bash
# Create Nginx
kubectl create deploy nginx-web-server --image nginx

# Expose Nginx
kubectl expose deploy nginx-web-server --port 80 --type NodePort
```

Get Nginx information
```
kubectl get services
kubectl get nodes -o wide
http://<Master-IP>:<NodePort>
or
http://<Node-IP>:<NodePort>
```

To view the nginx web server, execute the following over your preferred web browser:
```
> MASTER_IP:NODEPORT_SERVICE_PORT
http://192.168.1.205:30894/
OR
> WORKER_IP:NODEPORT_SERVICE_PORT
http://192.168.1.201:30894/
```