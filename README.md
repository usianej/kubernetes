# Kubernetes Home Lab Setup 

Welcome to the Kubernetes Home Lab repository! This repository is dedicated to helping you set up and manage a Kubernetes cluster from scratch using VirtualBox, Ubuntu, and Kubeadm, as well as exploring various aspects of Kubernetes.

## Table of Contents

- [Introduction](#introduction)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
  - [Setting up VirtualBox and Ubuntu](#setting-up-virtualbox-and-ubuntu)
  - [Initial Node Setup](#initial-node-setup)
  - [Containerd Runtime Setup and Configuration](#containerd-runtime-setup-and-configuration)
  - [Installing Kubeadm, Kubelet, and Kubectl](#installing-kubeadm-kubelet-and-kubectl)
  - [Control Plane Setup](#control-plane-setup)
  - [Initializing the Kubernetes Cluster](#initializing-the-kubernetes-cluster)
  - [Joining Nodes to the Cluster](#joining-nodes-to-the-cluster)
  - [Accessing Your Cluster](#accessing-your-cluster)
  - [Pod Network Setup](#pod-network-setup)
  - [System Checks](#system-checks)
- [Cluster Management](#cluster-management)
  - [Node Management](#node-management)
  - [Pod Management](#pod-management)
  - [Networking](#networking)
  - [Storage](#storage)
  - [Security](#security)
- [Advanced Topics](#advanced-topics)
  - [Monitoring and Logging](#monitoring-and-logging)
  - [CI/CD Integration](#cicd-integration)
  - [Scaling](#scaling)
- [Contributing](#contributing)
- [License](#license)

## Introduction

This repository provides comprehensive guidance on setting up and managing a Kubernetes cluster in your home lab environment. Whether you are a beginner or an experienced user, you'll find valuable information and step-by-step instructions for various Kubernetes operations.

## Prerequisites

Before you begin, ensure you have the following:

- [VirtualBox](https://www.virtualbox.org/) installed on your machine
- [Ubuntu ISO](https://releases.ubuntu.com/focal/) file for the virtual machines
- A system with at least 8GB of RAM and 4 CPU cores

## Installation

### Setting up VirtualBox and Ubuntu

1. **Download and Install VirtualBox:**
   - Download VirtualBox from the [official website](https://www.virtualbox.org/).
   - Follow the installation instructions for your operating system.
   
2. **Doenload ISO Image:**
   - Download Ubuntu Image from [ubuntu](https://releases.ubuntu.com/focal/).

2. **Create Virtual Machines:**
   - Open VirtualBox and create a new virtual machine for the master node.
   - Allocate at least 2GB of RAM and 2 CPU cores.
   - Attach the Ubuntu ISO file and install Ubuntu on the virtual machine.
   - Repeat the steps to create additional virtual machines for worker nodes.

---------------------------
### Initial Node Setup

- **Step1: Update cluster:**

```
sudo apt-get update
```
---

- **Step 2: Disable Swap**

```
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```
---

- **Step 3: Add Kernel Parameters**

```
sudo tee /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF
```
`sudo modprobe overlay`

`sudo modprobe br_netfilter`

```
sudo tee /etc/sysctl.d/kubernetes.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
```

**Apply sysctl params without reboot**

```
sudo sysctl --system
```

---

### Containerd Runtime Setup and Configuration

**[Option 1]**

`apt install curl -y`

```
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmour -o /etc/apt/trusted.gpg.d/docker.gpg

sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
```
```
sudo apt update
```

```
sudo apt install -y containerd.io
```
```
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
```
```
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
```
```
sudo systemctl restart containerd
sudo systemctl enable containerd
systemctl status containerd
```

**[Option 2]**

```
sudo apt update
```

```
sudo apt install -y containerd
```

```
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
```

`sudo systemctl restart containerd`

`sudo systemctl enable containerd`

---

### Installing Kubeadm, Kubelet, and Kubectl

```
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
```
sudo mkdir -p /etc/apt/keyrings/
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```
```
sudo apt-get update
apt-cache policy kubelet | head -n 20
```

> Install the required packages, if needed we can request specific version

```
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

**check status of the container** 

`sudo systemctl status kubelet.service`
`sudo systemctl status containerd.service`

> with no cluster configuration yet, kubelet will be crash looping. Bootstraping a cluster will help solve this 

---

### Control Plane Setup

```
wget https://docs.projectcalico.org/manifests/calico.yaml
```
```
kubeadm config print init-defaults | tee ClusterConfiguration.yaml
```

> Ensre you change the IP address to your Master/contol plane node IP

```bash
sed -i 's/ advertiseAddress: 1.2.3.4/ advertiseAddress: **10.10.4.5**/' ClusterConfiguration.yaml
```
```
sed -i 's/ criSocket: \/var\/run\/dockershim\.sock/ criSocket: \/run\/containerd\.sock/' ClusterConfiguration.yaml
```

```
cat <<EOF | cat >> ClusterConfiguration.yaml
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
EOF
```
### Initializing the Kubernetes Cluster

`sudo kubeadm init --config=ClusterConfiguration.yaml`

### Joining Nodes to the Cluster

After initializing (bootstrapping) the cluster with the cluster configuration, you will get an output as seen below. Take this output and run it on every node that you intend to add to the cluster.

```
kubeadm join 10.10.4.5:6443 --token abcdef.0123456789abcdef \
        --discovery-token-ca-cert-hash sha256:2ff9f8adae2ac45dd2619522964c3784e95d23cbbb07df0ca9b0a4ccbbe8e0db

```

### Accessing Your Cluster

> To start using your cluster, you need to run the following as a regular user:

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

> Note: CoreDNS will always remain pending until the pod network is up. It essentially has no network to exist on until then

### Pod Network Setup

Deploy yaml file for your pod network
```
kubectl apply -f calico.yaml
```
---
### System Checks
------------------------------------------------------------
> Look for the all the system pods and calico pods to change to Running. 

> The DNS pod won't start (pending) until the Pod network is deployed and Running.

`kubectl get pods --all-namespaces`


> Gives you output over time, rather than repainting the screen on each iteration.

`kubectl get pods --all-namespaces --watch`


> All system pods should be Running

`kubectl get pods --all-namespaces`


> Get a list of our current nodes, just the Control Plane Node Node...should be Ready.

`kubectl get nodes `
