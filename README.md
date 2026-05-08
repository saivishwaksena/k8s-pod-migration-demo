# Web Application Cluster Deployment and Controlled Pod Migration

## Project Overview

This project demonstrates deployment of a containerized web application on a Kubernetes cluster and controlled pod migration using `kubectl drain`.

The cluster was created using VMware Workstation with Ubuntu 22.04 virtual machines.

## Cluster Architecture

- 1 Master Node: `k8s-master`
- 2 Worker Nodes: `k8s-worker1`, `k8s-worker2`
- Kubernetes setup method: `kubeadm`
- Container runtime: `containerd`
- CNI plugin: Flannel
- Application: Nginx-based static web application
- Service exposure: NodePort

## Project Flow

1. Created three Ubuntu 22.04 VMs.
2. Installed containerd on all nodes.
3. Installed Kubernetes tools: kubeadm, kubelet, kubectl.
4. Initialized the master node using kubeadm.
5. Installed Flannel CNI.
6. Joined two worker nodes to the cluster.
7. Created a simple Nginx web application.
8. Built and pushed Docker image to Docker Hub.
9. Deployed the application using Kubernetes Deployment.
10. Exposed the application using NodePort.
11. Demonstrated controlled pod migration using `kubectl drain`.

## Kubernetes Cluster Verification

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide
