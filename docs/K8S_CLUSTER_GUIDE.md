# DevOps Internship - Complete Kubernetes Cluster Deployment Guide (From Zero to Hero)

**Name:** Zinab Mohamed Elsayed
**Task:** Kubernetes Cluster Setup, Configuration & Multi-Container Application Deployment

## Introduction & Overview

This document provides a comprehensive, step-by-step technical walkthrough of setting up a
2-node Kubernetes cluster (1 Master, 1 Worker) on Rocky Linux 9 from scratch, configuring
networking, managing cluster taints, and deploying the multi-container application
(PostgreSQL + Redis + Python app replicas + NGINX) into the `barq-assessment` namespace.

> Convention used in this guide: replace `<MASTER_IP>` with your master node's IP,
> `<TOKEN>` / `<HASH>` with the values printed by `kubeadm init`. Tokens expire after
> 24 hours by default.

## Environment

| Role | Hostname | OS | vCPU | RAM | Disk | IP |
|---|---|---|---|---|---|---|
| Master | k8s-master | Rocky Linux 9 | 2+ | 4 GB | 20 GB+ | `<MASTER_IP>` |
| Worker | k8s-worker1 | Rocky Linux 9 (full clone of master **before** `kubeadm init`) | 2 | 2 GB | 20 GB+ | `<WORKER_IP>` |

Cluster parameters:

- Pod network CIDR: `10.244.0.0/16` (Flannel default)
- CNI: Flannel
- Container runtime: containerd (with `SystemdCgroup = true`)
- Namespace: `barq-assessment`
- Application manifests (in this repo under `k8s/`): `namespace.yaml`, `secret.yaml`,
  `postgres.yaml`, `redis.yaml`, `app.yaml`, `nginx.yaml`

## VM Preparation

1. Install Rocky Linux 9 on the master VM (user `zinabelamir`, or your own user).
2. Create the worker as a **full clone** of the master **before** running `kubeadm init`:
   VirtualBox: right-click master name -> Manage -> Clone -> Full clone.
3. Set worker resources to **2 GB RAM** (minimum) and 2 vCPUs.
4. Power on both VMs and confirm they can reach each other and the internet:

```bash
ping -c 3 <MASTER_IP>
ping -c 3 8.8.8.8
ip addr show
```

[SCREENSHOT: VirtualBox clone settings and worker RAM set to 2 GB]

## 1. Prerequisites & System Configuration (Both Nodes)

Run on **both** master and worker before initializing Kubernetes.

### 1.1 Disable swap

Kubernetes requires swap to be off.

```bash
sudo swapoff -a
sudo sed -i.bak '/\sswap\s/s/^/#/' /etc/fstab
free -h
```

Expected: the `Swap:` line shows `0B` used.

[SCREENSHOT: `free -h` output showing swap disabled]

### 1.2 Load kernel modules and set sysctl

```bash
sudo modprobe overlay
sudo modprobe br_netfilter

sudo tee /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

Verify:

```bash
lsmod | grep -E 'overlay|br_netfilter'
sysctl net.bridge.bridge-nf-call-iptables net.ipv4.ip_forward
```

### 1.3 Set SELinux to permissive and update the system

```bash
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config
getenforce
sudo dnf update -y
```

Expected `getenforce` output: `Permissive`.

## 2. Installing Container Runtime & Kubernetes Tools (Both Nodes)

### 2.1 Install containerd

```bash
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf install -y containerd.io

sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
grep SystemdCgroup /etc/containerd/config.toml

sudo systemctl restart containerd
sudo systemctl enable containerd
sudo systemctl status containerd --no-pager
```

Expected: `SystemdCgroup = true` and containerd `active (running)`.

### 2.2 Add the Kubernetes yum repository (required on Rocky 9)

The `kubelet`/`kubeadm`/`kubectl` packages are not in the base repos, so add the
upstream repository first:

```bash
sudo tee /etc/yum.repos.d/kubernetes.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.30/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF
```

### 2.3 Install kubelet, kubeadm, kubectl

```bash
sudo dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable --now kubelet
kubectl version --client
kubeadm version
```

[SCREENSHOT: containerd and kubelet service status, kubectl client version]

## 3. Initializing the Master Node

Run **only on `k8s-master`**:

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

Save the `kubeadm join ... --token ... --discovery-token-ca-cert-hash ...` line from the
output. If you lose it later, regenerate with:

```bash
kubeadm token create --print-join-command
```

Configure `kubectl` for your regular user:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl get nodes
```

At this point the master is `NotReady` (expected until the CNI is installed).

[SCREENSHOT: `kubeadm init` success output and join command]

## 4. Installing the CNI Network Plugin (Flannel)

Run **on the master**:

```bash
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
kubectl get pods -n kube-system -w
kubectl get nodes
```

Expected: Flannel pods `Running`, master node becomes `Ready`.

[SCREENSHOT: Flannel pods Running in kube-system, `kubectl get nodes` showing master Ready]

## 5. Joining the Worker Node

Run **on `k8s-worker1`** (paste your real values):

```bash
sudo kubeadm join <MASTER_IP>:6443 --token <TOKEN> --discovery-token-ca-cert-hash <HASH>
```

If the join fails because of firewall rules, open the required ports or stop
firewalld for lab use:

```bash
sudo systemctl stop firewalld
sudo systemctl disable firewalld
```

Then verify **on the master**:

```bash
kubectl get nodes -o wide
```

[SCREENSHOT: worker join success output, `kubectl get nodes` showing both nodes Ready]

## 6. Verifying Node Readiness

```bash
kubectl get nodes
kubectl get pods -A -o wide
```

Expected: `k8s-master` and `k8s-worker1` both `Ready`, all `kube-system` pods `Running`.

[SCREENSHOT: `kubectl get nodes` showing Ready status for both nodes]

## 7. Managing Control Plane Taints

By default the control plane has a `NoSchedule` taint so workloads land on the worker.
Re-apply it explicitly (idempotent) and verify:

```bash
kubectl taint nodes k8s-master node-role.kubernetes.io/control-plane:NoSchedule
kubectl describe node k8s-master | grep -i taints
```

Expected: `Taints: node-role.kubernetes.io/control-plane:NoSchedule`.

To allow scheduling on the master again (optional, not used in this task):

```bash
kubectl taint nodes k8s-master node-role.kubernetes.io/control-plane:NoSchedule-
```

[SCREENSHOT: taint applied output and describe showing the taint]

## 8. Deploying the Multi-Container Application

The application stack is: PostgreSQL (1 replica) + Redis (1 replica) +
Python `barq-app` (3 replicas) + NGINX (2 replicas, NodePort `30080`),
all in namespace `barq-assessment`. The manifests live in `k8s/` in this repository.

Apply them **in order** (secret before postgres/app):

```bash
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/secret.yaml
kubectl apply -f k8s/postgres.yaml
kubectl apply -f k8s/redis.yaml
kubectl apply -f k8s/app.yaml
kubectl apply -f k8s/nginx.yaml

kubectl -n barq-assessment get pods -o wide -w
```

Notes:

- `k8s/app.yaml` uses image `docker.io/zinabelamir/barq-app:latest`. The cluster pulls
  it from Docker Hub, so the BONUS CD pipeline (`.github/workflows/cd.yml`) must have
  pushed it at least once, or `kubectl set image` must point at a tag you pushed.
- `k8s/secret.yaml` provides `POSTGRES_PASSWORD` for postgres and the app.
- `k8s/postgres.yaml` includes the `postgres-init` ConfigMap that creates the `records`
  table; `k8s/nginx.yaml` includes the `nginx-conf` ConfigMap and exposes NodePort `30080`.

Wait until everything is Ready, then confirm distribution across nodes:

```bash
kubectl -n barq-assessment get pods -o wide
kubectl -n barq-assessment rollout status deployment/barq-app --timeout=180s
kubectl -n barq-assessment rollout status deployment/postgres --timeout=120s
kubectl -n barq-assessment rollout status deployment/redis --timeout=120s
kubectl -n barq-assessment rollout status deployment/nginx --timeout=120s
```

[SCREENSHOT: `kubectl get pods -o wide` showing postgres, redis, barq-app replicas and nginx running]

## 9. Services & Endpoints Verification

```bash
kubectl -n barq-assessment get svc
kubectl -n barq-assessment get endpoints barq-app
kubectl -n barq-assessment get endpoints postgres
kubectl -n barq-assessment get endpoints redis
```

Expected services:

| Service | Type | Port |
|---|---|---|
| `postgres` | ClusterIP | 5432 |
| `redis` | ClusterIP | 6379 |
| `barq-app` | ClusterIP | 8080 |
| `nginx` | NodePort | 80 -> 30080 |

[SCREENSHOT: `kubectl get svc` and endpoints output]

## 10. Connectivity & Response Testing

### 10.1 Test the app ClusterIP service from inside the cluster

```bash
kubectl -n barq-assessment run temp-curl --rm -it --image=curlimages/curl --restart=Never -- \
  curl -s http://barq-app:8080/
```

Expected: JSON with `service: barq-api`, the configured `message`, and an `instance_id`.

### 10.2 Test health, readiness, records and counter

```bash
kubectl -n barq-assessment run temp-curl --rm -it --image=curlimages/curl --restart=Never -- \
  curl -s http://barq-app:8080/health; echo

kubectl -n barq-assessment run temp-curl --rm -it --image=curlimages/curl --restart=Never -- \
  curl -s http://barq-app:8080/ready; echo

kubectl -n barq-assessment run temp-curl --rm -it --image=curlimages/curl --restart=Never -- \
  curl -s -X POST -H 'Content-Type: application/json' \
  -d '{"title":"k8s smoke test"}' http://barq-app:8080/records; echo

kubectl -n barq-assessment run temp-curl --rm -it --image=curlimages/curl --restart=Never -- \
  curl -s http://barq-app:8080/counter; echo
```

`/ready` must report `"status": "ready"` with both `postgres` and `redis` ready.

### 10.3 Test NGINX from outside the cluster (host)

```bash
curl -s http://<MASTER_IP>:30080/; echo
curl -s http://<WORKER_IP>:30080/; echo
```

Expected: the same app JSON, routed through NGINX (NodePort `30080`).

[SCREENSHOT: curl responses showing HTTP 200 and app JSON from both paths]

## Troubleshooting Quick Reference

| Symptom | Likely cause | Fix |
|---|---|---|
| Node stays `NotReady` | Flannel not applied or pod CIDR mismatch | Re-apply Flannel; confirm `--pod-network-cidr=10.244.0.0/16` |
| `ImagePullBackOff` on `barq-app` | Image not pushed to Docker Hub yet | Run the CD pipeline once (`Actions -> cd`), or `podman push docker.io/zinabelamir/barq-app:latest` |
| `CrashLoopBackOff` on `barq-app` | Missing secret or DB unreachable | `kubectl -n barq-assessment get secret postgres-secret`; check `/ready` output |
| Worker cannot join | Firewall or expired token | Stop firewalld for lab; regenerate join command with `kubeadm token create --print-join-command` |
| Pods only on master | Taint removed | Re-apply `kubectl taint nodes k8s-master node-role.kubernetes.io/control-plane:NoSchedule` |

## Screenshot Checklist (add these where marked above)

1. VM clone settings + worker 2 GB RAM
2. `free -h` (swap off)
3. containerd + kubelet status
4. `kubeadm init` success + join command (redact token in shared docs if needed)
5. Flannel pods + master Ready
6. Worker join output + 2 nodes Ready
7. Taint describe output
8. `get pods -o wide` (all app pods Running)
9. `get svc` + endpoints
10. curl 200 responses (in-cluster + NodePort)
