# Web Application Cluster Deployment and Controlled Pod Migration

## Project Title

**Web Application Cluster Deployment and Controlled Pod Migration**

---

## Project Overview

This project demonstrates how to create a Kubernetes cluster using virtual machines, deploy a containerized web application, expose it using a NodePort service, and perform controlled pod migration using `kubectl drain`.

The main goal of this project is to show how Kubernetes maintains application availability even when one worker node is drained for maintenance.

---

## Key Features

- Created a 3-node Kubernetes cluster using VMware virtual machines
- Used `kubeadm` for Kubernetes cluster setup
- Used `containerd` as the container runtime
- Installed Flannel as the CNI plugin
- Created a simple Nginx-based web application
- Built and pushed Docker image to Docker Hub
- Deployed the application with 3 replicas
- Exposed the application using NodePort
- Demonstrated controlled pod migration using `kubectl drain`
- Verified that the web application remained accessible after pod migration

---

## Tools and Technologies Used

| Tool / Technology | Purpose |
|---|---|
| VMware Workstation | To create Ubuntu virtual machines |
| Ubuntu 22.04 | Operating system for all nodes |
| Kubernetes | Container orchestration |
| kubeadm | Kubernetes cluster setup |
| kubelet | Node agent for Kubernetes |
| kubectl | Kubernetes command-line tool |
| containerd | Container runtime used by Kubernetes |
| Flannel | Pod networking / CNI plugin |
| Docker | To build and push application image |
| Docker Hub | Container image registry |
| Nginx | Web server for static web application |
| GitHub | Project version control and documentation |

---

## Cluster Architecture

~~~text
Host Machine
   |
   | VMware Workstation
   |
   +------------------------------------------------+
   | Kubernetes Cluster                             |
   |                                                |
   |  k8s-master                                    |
   |  Role: Control Plane                           |
   |  IP: 192.168.19.139                            |
   |                                                |
   |  k8s-worker1                                   |
   |  Role: Worker Node                             |
   |  IP: 192.168.19.140                            |
   |                                                |
   |  k8s-worker2                                   |
   |  Role: Worker Node                             |
   |  IP: 192.168.19.141                            |
   +------------------------------------------------+
                |
                |
        Web Application Deployment
                |
        3 Replicas of Nginx Web App
                |
        NodePort Service: 30080
~~~

---

## Project Flow

~~~text
VM Creation
   ↓
Ubuntu 22.04 Installation
   ↓
Hostname and Network Configuration
   ↓
containerd Installation
   ↓
Kubernetes Tools Installation
   ↓
Master Node Initialization using kubeadm
   ↓
Flannel CNI Installation
   ↓
Worker Nodes Join the Cluster
   ↓
Web Application Creation
   ↓
Docker Image Build and Push
   ↓
Kubernetes Deployment and Service Creation
   ↓
NodePort Browser Access
   ↓
Controlled Pod Migration using kubectl drain
   ↓
Application Availability Verification
~~~

---

# Complete Implementation Steps

---

## 1. VM Setup

Three Ubuntu 22.04 virtual machines were created using VMware Workstation.

| Node Name | Role | IP Address |
|---|---|---|
| k8s-master | Master / Control Plane | 192.168.19.139 |
| k8s-worker1 | Worker Node | 192.168.19.140 |
| k8s-worker2 | Worker Node | 192.168.19.141 |

Recommended configuration:

| Node | RAM | CPU | Disk |
|---|---:|---:|---:|
| k8s-master | 2 GB or more | 2 cores | 30 GB |
| k8s-worker1 | 2 GB or more | 2 cores | 30 GB |
| k8s-worker2 | 2 GB or more | 2 cores | 30 GB |

Network mode used:

~~~text
VMware NAT Network
~~~

---

## 2. Set Hostnames

### On Master Node

~~~bash
sudo hostnamectl set-hostname k8s-master
sudo reboot
~~~

### On Worker Node 1

~~~bash
sudo hostnamectl set-hostname k8s-worker1
sudo reboot
~~~

### On Worker Node 2

~~~bash
sudo hostnamectl set-hostname k8s-worker2
sudo reboot
~~~

After reboot, verify hostname on each node:

~~~bash
hostname
~~~

Expected output:

~~~text
k8s-master
k8s-worker1
k8s-worker2
~~~

---

## 3. Configure `/etc/hosts`

Run on all three nodes:

~~~bash
sudo nano /etc/hosts
~~~

Add the following entries:

~~~text
192.168.19.139 k8s-master
192.168.19.140 k8s-worker1
192.168.19.141 k8s-worker2
~~~

Test connectivity:

~~~bash
ping -c 2 k8s-master
ping -c 2 k8s-worker1
ping -c 2 k8s-worker2
~~~

---

## 4. Disable Swap

Kubernetes requires swap to be disabled.

Run on all nodes:

~~~bash
sudo swapoff -a
~~~

Disable swap permanently:

~~~bash
sudo nano /etc/fstab
~~~

Comment the swap line by adding `#`.

Example:

~~~text
#/swap.img none swap sw 0 0
~~~

Verify:

~~~bash
free -h
~~~

Expected:

~~~text
Swap: 0B 0B 0B
~~~

---

## 5. Enable Required Kernel Modules

Run on all nodes:

~~~bash
sudo modprobe overlay
sudo modprobe br_netfilter
~~~

Make the modules load permanently:

~~~bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
~~~

Add Kubernetes networking settings:

~~~bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
~~~

Apply settings:

~~~bash
sudo sysctl --system
~~~

Verify:

~~~bash
lsmod | grep br_netfilter
lsmod | grep overlay
sysctl net.ipv4.ip_forward
~~~

Expected:

~~~text
net.ipv4.ip_forward = 1
~~~

---

## 6. Install containerd

Run on all nodes:

~~~bash
sudo apt update -o Acquire::ForceIPv4=true
sudo apt install -y containerd -o Acquire::ForceIPv4=true
~~~

Create containerd configuration:

~~~bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
~~~

Enable systemd cgroup driver:

~~~bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
~~~

Verify:

~~~bash
grep SystemdCgroup /etc/containerd/config.toml
~~~

Expected:

~~~text
SystemdCgroup = true
~~~

Restart and enable containerd:

~~~bash
sudo systemctl restart containerd
sudo systemctl enable containerd
~~~

Check status:

~~~bash
sudo systemctl status containerd
~~~

Expected:

~~~text
active (running)
~~~

---

## 7. Install Kubernetes Tools

Run on all nodes:

~~~bash
sudo apt update -o Acquire::ForceIPv4=true
sudo apt install -y apt-transport-https ca-certificates curl gpg -o Acquire::ForceIPv4=true
~~~

Create keyrings folder:

~~~bash
sudo mkdir -p /etc/apt/keyrings
~~~

Add Kubernetes repository key:

~~~bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --batch --yes --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
~~~

Add Kubernetes repository:

~~~bash
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
~~~

Update package list:

~~~bash
sudo apt update -o Acquire::ForceIPv4=true
~~~

Install Kubernetes components:

~~~bash
sudo apt install -y kubelet kubeadm kubectl -o Acquire::ForceIPv4=true
~~~

Hold versions:

~~~bash
sudo apt-mark hold kubelet kubeadm kubectl
~~~

Verify:

~~~bash
kubeadm version
kubectl version --client
kubelet --version
~~~

---

## 8. Disable IPv6 to Avoid Image Pull Issues

In this setup, Kubernetes image pulling failed because the VM network tried to use IPv6. IPv6 was disabled to force IPv4 communication.

Run on all nodes:

~~~bash
sudo cp /etc/default/grub /etc/default/grub.bak
sudo sed -i '/^GRUB_CMDLINE_LINUX=/ s/"$/ ipv6.disable=1"/' /etc/default/grub
sudo update-grub
sudo reboot
~~~

After reboot, verify:

~~~bash
cat /proc/cmdline | grep ipv6.disable
ip a | grep inet6
~~~

Expected:

~~~text
ipv6.disable=1
~~~

The second command should ideally give no output.

---

## 9. Configure Stable DNS

Run on all nodes:

~~~bash
sudo nano /etc/systemd/resolved.conf
~~~

Add or update these lines:

~~~text
DNS=8.8.8.8 1.1.1.1
FallbackDNS=8.8.4.4 9.9.9.9
DNSStubListener=yes
~~~

Restart services:

~~~bash
sudo systemctl restart systemd-resolved
sudo systemctl restart NetworkManager
sudo systemctl restart containerd
sudo systemctl restart kubelet
~~~

Verify DNS:

~~~bash
getent hosts registry.k8s.io
getent hosts ghcr.io
getent hosts pkg-containers.githubusercontent.com
ping -c 4 google.com
~~~

---

## 10. Pull Kubernetes Images on Master

Run only on the master node:

~~~bash
sudo kubeadm config images pull --kubernetes-version=v1.30.14
~~~

Expected output:

~~~text
[config/images] Pulled registry.k8s.io/kube-apiserver:v1.30.14
[config/images] Pulled registry.k8s.io/kube-controller-manager:v1.30.14
[config/images] Pulled registry.k8s.io/kube-scheduler:v1.30.14
[config/images] Pulled registry.k8s.io/kube-proxy:v1.30.14
[config/images] Pulled registry.k8s.io/coredns/coredns:v1.11.3
[config/images] Pulled registry.k8s.io/pause:3.9
[config/images] Pulled registry.k8s.io/etcd:3.5.15-0
~~~

---

## 11. Initialize Kubernetes Master

Run only on master:

~~~bash
sudo kubeadm init \
  --apiserver-advertise-address=192.168.19.139 \
  --pod-network-cidr=10.244.0.0/16
~~~

Expected output:

~~~text
Your Kubernetes control-plane has initialized successfully!
~~~

Configure kubectl for the normal user:

~~~bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
~~~

Check node:

~~~bash
kubectl get nodes
~~~

Expected before CNI installation:

~~~text
k8s-master   NotReady   control-plane
~~~

---

## 12. Install Flannel CNI

Run only on master:

~~~bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
~~~

Check pods:

~~~bash
kubectl get pods -A
~~~

Check node status:

~~~bash
kubectl get nodes
~~~

Expected:

~~~text
k8s-master   Ready   control-plane
~~~

---

## 13. Join Worker Nodes to Cluster

After `kubeadm init`, a join command is generated.

Example:

~~~bash
sudo kubeadm join 192.168.19.139:6443 --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
~~~

Run the join command on both worker nodes.

If the token expires, generate a new command on master:

~~~bash
kubeadm token create --print-join-command
~~~

Verify all nodes from master:

~~~bash
kubectl get nodes -o wide
~~~

Expected:

~~~text
NAME          STATUS   ROLES           INTERNAL-IP
k8s-master    Ready    control-plane   192.168.19.139
k8s-worker1   Ready    <none>          192.168.19.140
k8s-worker2   Ready    <none>          192.168.19.141
~~~

---

## 14. Verify Cluster Health

Run on master:

~~~bash
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl cluster-info
~~~

Expected:

- All nodes should be `Ready`
- CoreDNS pods should be `Running`
- Flannel pods should be `Running`
- kube-proxy pods should be `Running`

---

# Web Application Deployment

---

## 15. Install Docker on Master

Docker was used to build and push the web application image.

Run only on master:

~~~bash
sudo apt update -o Acquire::ForceIPv4=true
sudo apt install -y docker.io -o Acquire::ForceIPv4=true
~~~

Start Docker:

~~~bash
sudo systemctl start docker
sudo systemctl enable docker
~~~

Allow current user to run Docker:

~~~bash
sudo usermod -aG docker $USER
newgrp docker
~~~

Verify:

~~~bash
docker --version
docker ps
~~~

---

## 16. Create Web Application

Create project folder:

~~~bash
mkdir -p ~/k8s-pod-migration-demo/webapp
cd ~/k8s-pod-migration-demo/webapp
~~~

Create `index.html`:

~~~bash
nano index.html
~~~

Add:

~~~html
<!DOCTYPE html>
<html>
<head>
  <title>Kubernetes Pod Migration Demo</title>
  <style>
    body {
      font-family: Arial, sans-serif;
      background: linear-gradient(135deg, #0f172a, #2563eb);
      color: white;
      text-align: center;
      padding-top: 100px;
    }
    .card {
      background: rgba(255, 255, 255, 0.12);
      padding: 40px;
      margin: auto;
      width: 70%;
      border-radius: 20px;
      box-shadow: 0 0 25px rgba(0,0,0,0.3);
    }
    h1 {
      font-size: 42px;
    }
    p {
      font-size: 20px;
    }
    .badge {
      margin-top: 20px;
      display: inline-block;
      padding: 10px 20px;
      border-radius: 999px;
      background: #22c55e;
      color: #052e16;
      font-weight: bold;
    }
  </style>
</head>
<body>
  <div class="card">
    <h1>Web Application Cluster Deployment</h1>
    <p>Controlled Pod Migration using Kubernetes</p>
    <p>1 Master Node + 2 Worker Nodes</p>
    <div class="badge">Application Running Successfully</div>
  </div>
</body>
</html>
~~~

Create Dockerfile:

~~~bash
nano Dockerfile
~~~

Add:

~~~dockerfile
FROM nginx:alpine

COPY index.html /usr/share/nginx/html/index.html

EXPOSE 80
~~~

Verify:

~~~bash
ls -l
~~~

Expected:

~~~text
Dockerfile
index.html
~~~

---

## 17. Build Docker Image

Replace `<dockerhub-username>` with your Docker Hub username.

~~~bash
cd ~/k8s-pod-migration-demo/webapp
docker build -t <dockerhub-username>/k8s-pod-migration-demo:v1 .
~~~

Example:

~~~bash
docker build -t saivishwaksena/k8s-pod-migration-demo:v1 .
~~~

Verify image:

~~~bash
docker images
~~~

---

## 18. Test Docker Container Locally

Run:

~~~bash
docker run -d --name webapp-test -p 8080:80 <dockerhub-username>/k8s-pod-migration-demo:v1
~~~

Test:

~~~bash
curl http://localhost:8080
~~~

Stop test container:

~~~bash
docker stop webapp-test
docker rm webapp-test
~~~

---

## 19. Push Docker Image to Docker Hub

Login:

~~~bash
docker login
~~~

Push image:

~~~bash
docker push <dockerhub-username>/k8s-pod-migration-demo:v1
~~~

Expected output:

~~~text
v1: digest: sha256:...
~~~

Verify by pulling image:

~~~bash
docker pull <dockerhub-username>/k8s-pod-migration-demo:v1
~~~

---

## 20. Create Kubernetes YAML Files

Create folder:

~~~bash
mkdir -p ~/k8s-pod-migration-demo/k8s
cd ~/k8s-pod-migration-demo/k8s
~~~

Create `deployment.yaml`:

~~~bash
nano deployment.yaml
~~~

Add:

~~~yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
        - name: webapp-container
          image: <dockerhub-username>/k8s-pod-migration-demo:v1
          ports:
            - containerPort: 80
~~~

Create `service.yaml`:

~~~bash
nano service.yaml
~~~

Add:

~~~yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp-service
spec:
  type: NodePort
  selector:
    app: webapp
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
~~~

---

## 21. Deploy Application on Kubernetes

Run on master:

~~~bash
cd ~/k8s-pod-migration-demo/k8s
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
~~~

Check deployment:

~~~bash
kubectl get deployments
kubectl get pods -o wide
kubectl get svc
~~~

Expected:

~~~text
webapp-deployment   3/3
webapp-service      NodePort   80:30080/TCP
~~~

---

## 22. Access Web Application

Open in browser:

~~~text
http://192.168.19.139:30080
http://192.168.19.140:30080
http://192.168.19.141:30080
~~~

Since NodePort exposes the service on all nodes, the application can be accessed using any node IP with port `30080`.

Test using terminal:

~~~bash
curl http://192.168.19.139:30080
curl http://192.168.19.140:30080
curl http://192.168.19.141:30080
~~~

---

# Controlled Pod Migration Demo

---

## 23. Check Pod Placement Before Migration

Run on master:

~~~bash
kubectl get pods -o wide
~~~

This shows which worker node each application pod is running on.

Optional watch command:

~~~bash
watch -n 2 kubectl get pods -o wide
~~~

---

## 24. Drain Worker Node

Drain worker1:

~~~bash
kubectl drain k8s-worker1 --ignore-daemonsets --delete-emptydir-data
~~~

Expected behavior:

- Worker1 is cordoned
- Pods running on worker1 are evicted
- Kubernetes creates replacement pods on another available worker node
- Application remains available through NodePort

Check pods:

~~~bash
kubectl get pods -o wide
~~~

Check nodes:

~~~bash
kubectl get nodes
~~~

Expected:

~~~text
k8s-worker1   Ready,SchedulingDisabled
~~~

---

## 25. Verify Application After Migration

Open in browser:

~~~text
http://192.168.19.139:30080
http://192.168.19.140:30080
http://192.168.19.141:30080
~~~

Even after draining `k8s-worker1`, the URL using worker1 IP may still work:

~~~text
http://192.168.19.140:30080
~~~

This happens because NodePort uses Kubernetes service networking and kube-proxy to forward traffic to healthy pods on available nodes.

---

## 26. Bring Worker Node Back

After the demo, uncordon worker1:

~~~bash
kubectl uncordon k8s-worker1
~~~

Verify:

~~~bash
kubectl get nodes
~~~

Expected:

~~~text
k8s-worker1   Ready
~~~

---

# GitHub Project Structure

~~~text
k8s-pod-migration-demo/
├── webapp/
│   ├── index.html
│   └── Dockerfile
├── k8s/
│   ├── deployment.yaml
│   └── service.yaml
├── screenshots/
│   └── .gitkeep
└── README.md
~~~

Create screenshots folder:

~~~bash
cd ~/k8s-pod-migration-demo
mkdir -p screenshots
touch screenshots/.gitkeep
~~~

---

# Push Project to GitHub

Configure Git:

~~~bash
git config --global user.name "saivishwaksena"
git config --global user.email "saivishwaksena77@gmail.com"
~~~

Initialize Git repository:

~~~bash
cd ~/k8s-pod-migration-demo
git init
git add .
git commit -m "Initial commit: Kubernetes pod migration demo"
~~~

Add GitHub remote:

~~~bash
git branch -M main
git remote add origin https://github.com/saivishwaksena/k8s-pod-migration-demo.git
~~~

Push:

~~~bash
git push -u origin main
~~~

GitHub does not support password authentication from terminal. Use a GitHub Personal Access Token as the password.

---

# Screenshots to Add

Add screenshots for proof:

1. VMware 3 VM setup
2. Hostname and IP verification
3. `kubectl get nodes -o wide`
4. `kubectl get pods -A -o wide`
5. Docker image in Docker Hub
6. `kubectl get deployments`
7. `kubectl get svc`
8. Web application running in browser
9. Pods before drain
10. Drain command output
11. Pods after drain
12. Browser access after migration
13. GitHub repository page

---

# Problems Faced and Fixes

## Problem 1: `apt update` was slow

Fix:

~~~bash
sudo apt update -o Acquire::ForceIPv4=true
~~~

---

## Problem 2: Kubernetes image pull failed due to IPv6

Error example:

~~~text
dial tcp [2600:1901:...]:443: connect: network is unreachable
~~~

Fix:

~~~bash
sudo sed -i '/^GRUB_CMDLINE_LINUX=/ s/"$/ ipv6.disable=1"/' /etc/default/grub
sudo update-grub
sudo reboot
~~~

---

## Problem 3: Flannel ImagePullBackOff due to DNS timeout

Error example:

~~~text
lookup ghcr.io on 127.0.0.53:53: i/o timeout
~~~

Fix:

~~~bash
sudo nano /etc/systemd/resolved.conf
~~~

Added:

~~~text
DNS=8.8.8.8 1.1.1.1
FallbackDNS=8.8.4.4 9.9.9.9
DNSStubListener=yes
~~~

Restarted services:

~~~bash
sudo systemctl restart systemd-resolved
sudo systemctl restart NetworkManager
sudo systemctl restart containerd
sudo systemctl restart kubelet
~~~

---

## Problem 4: GitHub password authentication failed

Error:

~~~text
Password authentication is not supported for Git operations.
~~~

Fix:

Used GitHub Personal Access Token instead of normal GitHub password.

---

# Important Commands Summary

Check all nodes:

~~~bash
kubectl get nodes -o wide
~~~

Check all pods:

~~~bash
kubectl get pods -A -o wide
~~~

Check application pods:

~~~bash
kubectl get pods -o wide
~~~

Check service:

~~~bash
kubectl get svc
~~~

Drain worker node:

~~~bash
kubectl drain k8s-worker1 --ignore-daemonsets --delete-emptydir-data
~~~

Bring worker node back:

~~~bash
kubectl uncordon k8s-worker1
~~~

Delete application:

~~~bash
kubectl delete -f k8s/deployment.yaml
kubectl delete -f k8s/service.yaml
~~~

Redeploy application:

~~~bash
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
~~~

---

# Final Result

The project successfully demonstrates:

- Kubernetes cluster creation using 3 Ubuntu virtual machines
- Web application containerization using Docker
- Docker image push to Docker Hub
- Kubernetes Deployment with 3 replicas
- NodePort-based browser access
- Controlled pod migration using `kubectl drain`
- Continued application availability after draining a worker node

---

# Conclusion

This project shows how Kubernetes helps maintain application availability during node maintenance. When a worker node is drained, Kubernetes safely evicts the pods and reschedules them on another available worker node. The NodePort service continues to route traffic to healthy pods, so the web application remains accessible even after pod migration.

---

# References

- Kubernetes Documentation
- Docker Documentation
- Flannel CNI Documentation
- Ubuntu 22.04 Documentation
