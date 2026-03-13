# High Availability Multi-Master Kubernetes Cluster Deployment

![Build Status](https://img.shields.io/badge/build-passing-brightgreen) ![License](https://img.shields.io/badge/license-MIT-blue) ![Kubernetes Version](https://img.shields.io/badge/kubernetes-v1.35-blue)

## Project Overview
This repository provides the documentation and automation scripts necessary to deploy a production-grade, Highly Available (HA) Kubernetes cluster. By utilizing multiple master nodes situated behind HAProxy and Keepalived load balancers, this architecture eliminates the control plane as a single point of failure, ensuring your containerized applications remain resilient and continuously accessible.

## Architecture Overview
`[Insert Architecture Diagram Here]`

This multi-master deployment consists of three distinct node roles:
1. **Load Balancers (HAProxy + Keepalived):** Two nodes configured in an active/passive setup to provide a single Virtual IP (VIP) that routes traffic reliably to the available Kubernetes API servers.
2. **Control Plane (Master Nodes):** Three master nodes that share the cluster state and run the core Kubernetes components. 
3. **Worker Nodes:** Three worker nodes designated for scheduling and executing your application workloads.

## Prerequisites
* **Operating System:** Ubuntu/Debian-based Linux distribution.
* **Privileges:** Root or `sudo` access on all virtual machines.
* **Network Configuration:** Eight VMs with statically assigned IPs as defined below:

| Role | Hostname | IP Address |
| :--- | :--- | :--- |
| **Load Balancer 1** | `lb-01` | `10.1.121.x` *(Assign locally)* |
| **Load Balancer 2** | `lb-02` | `10.1.121.y` *(Assign locally)* |
| **Virtual IP (VIP)** | `k8s-vip` | `10.1.121.2` *(Note: Alternative configurations may use 10.1.121.100)* |
| **Master Node 1** | `k8s-master-01` | `10.1.121.11`  |
| **Master Node 2** | `k8s-master-02` | `10.1.121.12`  |
| **Master Node 3** | `k8s-master-03` | `10.1.121.13`  |
| **Worker Node 1** | `k8s-worker-01` | `10.1.121.14`  |
| **Worker Node 2** | `k8s-worker-02` | `10.1.121.15`  |
| **Worker Node 3** | `k8s-worker-03` | `10.1.121.16`  |


## Step-by-Step Deployment Guide

### Phase 1: Set Up Load Balancer Nodes (LoadBalancer1 & LoadBalancer2)
Execute these steps on **both** load balancer nodes to configure the VIP and API server routing.

**1. Install Dependencies**
```
apt update && apt install -y keepalived haproxy 
```

**2. Configure Keepalived Health Check** Create the health check script to ensure traffic is only routed to healthy API servers:

```
cat >> /etc/keepalived/check_apiserver.sh <<EOF
#!/bin/sh
errorExit () {
  echo "*** \$@" 1>&2
  exit 1
}
curl --silent --max-time 2 --insecure https://localhost:6443/ -o /dev/null || errorExit "Error GET https://localhost:6443/"
if ip addr | grep -q 10.1.121.2; then
  curl --silent --max-time 2 --insecure [https://10.1.121.2:6443/](https://10.1.121.2:6443/) -o /dev/null || errorExit "Error GET [https://10.1.121.2:6443/](https://10.1.121.2:6443/)"
fi
EOF

chmod +x /etc/keepalived/check_apiserver.sh 
```

**3. Apply Keepalived Configuration** Create the configuration file defining the VIP (`10.1.121.2`):

```
cat >> /etc/keepalived/keepalived.conf <<EOF
vrrp_script check_apiserver {
  script "/etc/keepalived/check_apiserver.sh"
  interval 3
  timeout 10
  fall 5
  rise 2
  weight -2
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens160
    virtual_router_id 1
    priority 100
    advert_int 5
    authentication {
        auth_type PASS
        auth_pass mysecret
    }
    virtual_ipaddress {
        10.1.121.2
    }
    track_script {
        check_apiserver
    }
}
EOF

systemctl enable --now keepalived 
```

**4. Configure HAProxy** Update the HAProxy configuration to balance traffic across the three master nodes:

```
cat >> /etc/haproxy/haproxy.cfg <<EOF
frontend kubernetes-frontend
  bind *:6443
  mode tcp
  option tcplog
  default_backend kubernetes-backend

backend kubernetes-backend
  option httpchk GET /healthz
  http-check expect status 200
  mode tcp
  option ssl-hello-chk
  balance roundrobin
    server kmaster1 10.1.121.11:6443 check fall 3 rise 2
    server kmaster2 10.1.121.12:6443 check fall 3 rise 2
    server kmaster3 10.1.121.13:6443 check fall 3 rise 2
EOF

systemctl enable haproxy && systemctl restart haproxy 
```

### Phase 2: Pre-requisites on all Kubernetes Nodes (Masters + Workers)

Run the following setup script on **every** Master and Worker node. Create the file and execute it:

```
touch kubernetes-cluster-setup.sh
chmod +x kubernetes-cluster-setup.sh
nano kubernetes-cluster-setup.sh 
```

Paste this script into the file:
```
#!/bin/bash 
set -e
echo "[Step 1] Updating /etc/hosts ..."
cat <<EOF | sudo tee -a /etc/hosts
192.168.172.51        k8s-master-01
192.168.172.52        k8s-worker-01
192.168.172.53        k8s-worker-02
192.168.172.54        k8s-worker-03
EOF

echo "[Step 2] Disabling swap ..."
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

echo "[Step 3] Configuring sysctl ..."
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.ipv4.ip_forward=1
EOF
sudo sysctl --system
sysctl net.ipv4.ip_forward

echo "[Step 4] Installing Kubernetes packages ..."
sudo apt-get update -y
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
sudo mkdir -p /etc/apt/keyrings
curl -fsSL [https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key](https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key) | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] [https://pkgs.k8s.io/core:/stable:/v1.35/deb/](https://pkgs.k8s.io/core:/stable:/v1.35/deb/) /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update -y
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

echo "[Step 5] Removing old container runtimes ..."
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do      
    sudo apt-get remove -y $pkg || true 
done 

echo "[Step 6] Installing containerd runtime ..."
sudo apt-get update -y
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL [https://download.docker.com/linux/ubuntu/gpg](https://download.docker.com/linux/ubuntu/gpg) -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
[https://download.docker.com/linux/ubuntu](https://download.docker.com/linux/ubuntu) $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
    sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update -y
sudo apt-get install -y containerd.io
sudo mkdir -p /etc/containerd
sudo sh -c "containerd config default > /etc/containerd/config.toml"
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

echo "[Step 7] Restarting services ..."
sudo systemctl restart containerd.service
sudo systemctl restart kubelet.service
sudo systemctl enable kubelet.service
echo "[✔] Worker node setup completed. Now run 'kubeadm join ...' from the master node output."
```

### Phase 3: Initialize the Control Plane (1st Master Node)

Execute this exclusively on the **1st Master Node** (`10.1.121.11`) to bootstrap the cluster using the VIP endpoint:


```
sudo kubeadm config images pull
sudo kubeadm init --control-plane-endpoint="10.1.121.2:6443" --upload-certs --apiserver-advertise-address=10.1.121.11 --pod-network-cidr=10.10.0.0/16 
```

Once complete, securely store the generated `kubeadm join` commands outputted in the terminal.

### Phase 4: Join Additional Master Nodes

Join the remaining master nodes to the control plane. Make sure to update the `--apiserver-advertise-address` flag with the specific IP of the node you are currently operating on.

On the 2nd Master Node (`10.1.121.12`):

```
sudo kubeadm join 10.1.121.2:6443 \
  --token 01pkyg.mnx7e7ym3pnjo8bl \
  --discovery-token-ca-cert-hash sha256:9cf417536fc1f57f3ac8ec89a7e2e46080dcfc7af64218d18638ca3c27108672 \
  --control-plane --certificate-key d4465572b7e8da91660ebb8a71edefa6f41d6e28d4e958811ede795e92950030 \
  --apiserver-advertise-address=10.1.121.12 
```

On the 3rd Master Node (`10.1.121.13`):

```
sudo kubeadm join 10.1.121.2:6443 \
  --token 01pkyg.mnx7e7ym3pnjo8bl \
  --discovery-token-ca-cert-hash sha256:9cf417536fc1f57f3ac8ec89a7e2e46080dcfc7af64218d18638ca3c27108672 \
  --control-plane --certificate-key d4465572b7e8da91660ebb8a71edefa6f41d6e28d4e958811ede795e92950030 \
  --apiserver-advertise-address=10.1.121.13 
```

### Phase 5: Join Worker Nodes

Run the worker join command on all designated worker nodes (`10.1.121.14`, `.15`, `.16`):

```
sudo kubeadm join 10.1.121.2:6443 \
  --token 01pkyg.mnx7e7ym3pnjo8bl \
  --discovery-token-ca-cert-hash sha256:9cf417536fc1f57f3ac8ec89a7e2e46080dcfc7af64218d18638ca3c27108672
```

## Validation

To verify the cluster is fully operational, run the following commands from any of your Master nodes:

```
# Verify all nodes have successfully joined and report as "Ready"
kubectl get nodes -o wide

# Verify system pods (CoreDNS, API Server, Kube-Proxy) are running
kubectl get pods -n kube-system
```
