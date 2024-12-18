# HA-K8s-Cluster-Stacked-etcd-HAProxy
A highly available Kubernetes cluster setup with a stacked etcd topology and HAProxy for load balancing. 

![HA K8 with Stacked etcd architecture](https://github.com/user-attachments/assets/46c73c04-d147-4f06-afda-1ba71b54d558)

## Features

- **High Availability**: Achieved through the use of stacked etcd and HAProxy for efficient load balancing.
- **Scalability**: Easily extend the setup by adding more control plane and worker nodes.
- **Fault Tolerance**: Ensures no single point of failure for the Kubernetes control plane.
- **Optimized for Production**: Designed for real-world production environments where uptime and reliability are critical.

## 🧠 Quorum in etcd

In a highly available etcd cluster, Quorum is the minimum number of nodes needed to agree on changes to maintain data consistency. It is crucial for ensuring the cluster’s reliability and avoiding issues like data inconsistency or split-brain

### 🚨 Why Quorum is Critical:
For **etcd** to function correctly:
- It requires a majority of nodes in the cluster to agree on any operation before it can be committed (write, delete, or update).
- For example:
   - In a **3-node** cluster: `(3 / 2) + 1 = 2`. So, at least **2 nodes** need to agree for the cluster to maintain a quorum.
   - In a **5-node** cluster: `(5 / 2) + 1 = 3`. So, at least **3 nodes** need to agree to form the quorum.
- If a node becomes unavailable (e.g., fails or is disconnected), the remaining nodes must still form a quorum to ensure the cluster remains functional.

## ⚙️ Architecture Requirements
*Hardware and Network Setup for High Availability*

- **Control Plane Nodes**: At least **3 control plane nodes** to maintain quorum.
  - **Minimum Specification**: **4 GB RAM** and **2 vCPUs** per node.
- **Worker Nodes**: At least **2 worker nodes** for handling pod workloads.
  - **Minimum Specification**: **4 GB RAM** and **2 vCPUs** per node.
- **Load Balancer**: 1 **VM** to implement **HAProxy** for load balancing between the control plane nodes.
- **Networking**: All components (control plane nodes, worker nodes, and load balancer) must reside within a **single VPC** since **internal IPs** will be used for communication between nodes.

## 🛠️ Implementation
### Setting up Loadbalancer
#### 1. Install HAProxy
First, update the package list and install HAProxy on your load balancer node:
```bash
sudo apt-get update
```
```bash
sudo apt-get install -y haproxy
```
#### 2. Configure HAProxy Config File
Open HAproxy config file:
```bash
sudo vi /etc/haproxy/haproxy.cfg
```
Add the following configurations to file:
```bash
frontend kubernetes-frontend
    bind *:6443
    option tcplog
    mode tcp
    default_backend kubernetes-backend

backend kubernetes-backend
    mode tcp
    balance roundrobin
    option tcp-check
    server controlplane1 <CONTROL_PLANE_1_PRIVATE_IP>:6443 check
    server controlplane2 <CONTROL_PLANE_2_PRIVATE_IP>:6443 check
    server controlplane3 <CONTROL_PLANE_3_PRIVATE_IP>:6443 check
```
#### 3. Restart the HAProxy 
To reflect the current changes in the configuration:  
```bash
sudo systemctl restart haproxy
```

### Setting UP k8 Cluster
Make sure to open the following ports:
- Master Node Ports: 2379,6443,10250,10251,10252
- Worker Node Ports: 10250,30000–32767.
- HAproxy node Ports: 6443
#### 1. Kubernetes Cluster Setup Script
This script is a common setup for all servers (Control Plane and Worker Nodes) in a Kubernetes cluster. It configures the system by disabling swap, loading necessary kernel modules, applying sysctl parameters, and installing the required dependencies, including CRI-O as the container runtime and Kubernetes components (kubelet, kubectl, kubeadm).

- Make sure to elevate the user privillages:
```bash
sudo -i
```
- Make an .sh file on all nodes:
 ```bash
vi script.sh
```

- This script sets variables to ensure Kubernetes (v1.30) and CRI-O (v1.30) versions match for compatibility during installation. It disables  swap (swapoff -a) to ensure 
  Kubernetes functions correctly, as it requires memory management without swap.
  
  ```bash
   set -euxo pipefail

   # Kubernetes Variable Declaration
   KUBERNETES_VERSION="v1.30"
   CRIO_VERSION="v1.30"
   KUBERNETES_INSTALL_VERSION="1.30.0-1.1"

   # Disable swap
   sudo swapoff -a

  # Keeps the swap off during reboot
  (crontab -l 2>/dev/null; echo "@reboot /sbin/swapoff -a") | crontab - || true

   ```
-  These instructions create the k8s.conf file to load the overlay and br_netfilter modules at boot. The overlay module allows containers to use a virtual network, while 
   br_netfilter enables proper communication between containers through network filtering.
    ```bash
    # Create the .conf file to load the modules at bootup
    cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
    overlay
    br_netfilter
    EOF

   sudo modprobe overlay
   sudo modprobe br_netfilter

    ```
-  This code sets sysctl parameters for Kubernetes networking, enabling IP forwarding and configuring bridge networks to use iptables.
   ```bash
   
   cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
   net.bridge.bridge-nf-call-iptables  = 1
   net.bridge.bridge-nf-call-ip6tables = 1
   net.ipv4.ip_forward                 = 1
   EOF

   
   sudo sysctl --system

   ```
-  This command updates the package list and installs necessary tools (apt-transport-https, ca-certificates, curl, gpg).
   ```bash
   sudo apt-get update -y
   sudo apt-get install -y apt-transport-https ca-certificates curl gpg

   ```
-  These instructions Install CRI-O Runtime and make sure it starts automatically at boot.
   ```bash
    # Install CRI-O Runtime
    sudo apt-get update -y
    sudo apt-get install -y software-properties-common curl apt-transport-https ca-certificates

    # Fetch and save the CRI-O GPG key
    curl -fsSL https://pkgs.k8s.io/addons:/cri-o:/stable:/$CRIO_VERSION/deb/Release.key |
    gpg --dearmor -o /etc/apt/keyrings/cri-o-apt-keyring.gpg

   # Add the CRI-O repository to APT sources
   echo "deb [signed-by=/etc/apt/keyrings/cri-o-apt-keyring.gpg] https://pkgs.k8s.io/addons:/cri-o:/stable:/$CRIO_VERSION/deb/ /" |
    tee /etc/apt/sources.list.d/cri-o.list

   # Update the package lists
   sudo apt-get update -y

   # Install CRI-O
   sudo apt-get install -y cri-o

   # Enable and start CRI-O service
   sudo systemctl daemon-reload
   sudo systemctl enable crio --now
   sudo systemctl start crio.service

   echo "CRI runtime installed successfully"

   ```
- These instructions Install Kubernetes Components
  ```bash
   # Install kubelet, kubectl, and kubeadm
   curl -fsSL https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/Release.key |
   gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

   # Add the Kubernetes repository to APT sources
   echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/$KUBERNETES_VERSION/deb/ /" |
    tee /etc/apt/sources.list.d/kubernetes.list

   # Update package lists again
   sudo apt-get update -y

   # Install kubelet, kubectl, and kubeadm with specific versions
   sudo apt-get install -y kubelet="$KUBERNETES_INSTALL_VERSION" kubectl="$KUBERNETES_INSTALL_VERSION" kubeadm="$KUBERNETES_INSTALL_VERSION"

   # Prevent automatic updates for kubelet, kubeadm, and kubectl
   sudo apt-mark hold kubelet kubeadm kubectl

  ```
- These instruction Retrieve Local IP and add it to kubelet configuration for kubelet to know which ip to use when talking to API server.
   ```bash
    # Install jq, a command-line JSON processor
    sudo apt-get install -y jq
   # Retrieve the local IP address of the eth0 interface and set it for kubelet
   local_ip="$(ip --json addr show ens4 | jq -r '.[0].addr_info[] | select(.family == "inet") | .local')"

   # Write the local IP address to the kubelet default configuration file
   cat > /etc/default/kubelet << EOF
   KUBELET_EXTRA_ARGS=--node-ip=$local_ip
   EOF


   ```
#### 2. Initialize the First Control plane
- Use the following command to initialize kubeadm on anyone of the control plane node
 ```bash
sudo kubeadm init --control-plane-endpoint "<LOADBALANCER_PRIVATE_IP>:6443" --upload-certs --pod-network-cidr 192.168.0.0/16
 ```
- After successful control plane initialization , we will get output :
![image](https://github.com/user-attachments/assets/1e735fc0-9ee0-4ed7-848b-d3dd576bdddd)

  
- Before adding more control planes and workers to cluser , configure user enviroment to interact with k8 cluster
   ```bash
  mkdir -p "$HOME"/.kube
  sudo cp -i /etc/kubernetes/admin.conf "$HOME"/.kube/config
  sudo chown "$(id -u)":"$(id -g)" "$HOME"/.kube/config
   ```
- Install Calico Network Plugin Network
   ```bash
   kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
   ```
#### 3. Add More Controlplane Nodes to CLuster
- Execute this particular command provided by first control plane on other nodes to join the cluster
  ![image](https://github.com/user-attachments/assets/707fa3d9-7387-4f03-a689-8da36b766e4d)
- again configure user enviroment to interact with k8 cluster
  ```bash
  mkdir -p "$HOME"/.kube
  sudo cp -i /etc/kubernetes/admin.conf "$HOME"/.kube/config
  sudo chown "$(id -u)":"$(id -g)" "$HOME"/.kube/config
   ```
#### 4. Add Worker node to CLuster
- Execute this particular command provided by first control plane on other nodes to join the cluster
  
  ![image](https://github.com/user-attachments/assets/c81abe8a-45c3-4fba-bc57-906a62af95c0)

- again configure user enviroment to interact with k8 cluster
  ```bash
  mkdir -p "$HOME"/.kube
  sudo cp -i /etc/kubernetes/admin.conf "$HOME"/.kube/config
  sudo chown "$(id -u)":"$(id -g)" "$HOME"/.kube/config
   ```
### 5. Verify Cluster Status 
- To see all nodes, execute this command in any of the  :
   ```bash
   kubectl get nodes
   ```
- if all control planes and workers are in ready state, it means  cluster is up and running.
  
  ![image](https://github.com/user-attachments/assets/58d56ddf-775b-4cdb-9c6f-21a503edb620)
### Testing HA Cluster
- Test the cluster by stopping any number of control plane , you will observer cluster is running smoothly even when a number of control planes are down ** Remember to follow quorom while testing it, beacuse if running control planes are less than the number desired to maintain High Availibilty, the whole cluster will go down

 







