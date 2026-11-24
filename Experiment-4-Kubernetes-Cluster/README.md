# Experiment 4: Kubernetes Cluster Orchestration

[![Course](https://img.shields.io/badge/Course-Cloud%20Computing-blue.svg)](#)
[![Experiment](https://img.shields.io/badge/Experiment-Lab%204-purple.svg)](#)
[![Cluster](https://img.shields.io/badge/Cluster-KinD%20(1%20Control%20%2B%202%20Workers)-orange.svg)](#)
[![Orchestration](https://img.shields.io/badge/Orchestration-Kubernetes%20v1.37.0-326CE5.svg?logo=kubernetes&logoColor=white)](#)
[![Workload](https://img.shields.io/badge/Workload-NGINX%20Web%20Server-009639.svg?logo=nginx&logoColor=white)](#)
[![Status](https://img.shields.io/badge/Status-Completed-brightgreen.svg)](#)

---

## Executive Summary

This experiment investigates container orchestration, declarative cluster management, multi-node pod scheduling, horizontal pod scaling, and automated self-healing fault tolerance using **Kubernetes (K8s)** running locally on Windows via **KinD (Kubernetes IN Docker)** and **Docker Desktop with WSL 2**.

A multi-node cluster topology comprising one control-plane node (`kind-control-plane`) and two dedicated worker nodes (`kind-worker`, `kind-worker2`) was declaratively provisioned. An NGINX web server deployment was orchestrated across the cluster, scaled horizontally from 3 to 6 replicas (achieving a balanced 50/50 distribution across worker nodes), and tested for high-availability self-healing by forcefully deleting active pods to verify automated zero-touch state reconciliation.

### Key Takeaway

> **KinD on Windows with WSL 2 provides a lightweight, deterministic environment for multi-node container orchestration. The Kubernetes declarative control loop automates pod scheduling, achieves balanced 50/50 horizontal load distribution across worker nodes, and executes autonomous self-healing within 5 seconds of container termination without manual intervention.**

---

## Table of Contents

1. [Objectives](#1-objectives)
2. [Architectural Topology](#2-architectural-topology)
3. [Environment Specifications & Tools](#3-environment-specifications--tools)
4. [Cluster Configuration Manifest (`kind-config.yaml`)](#4-cluster-configuration-manifest-kind-configyaml)
5. [Step-by-Step Procedure & Execution Workflow](#5-step-by-step-procedure--execution-workflow)
6. [Empirical Results & Verification](#6-empirical-results--verification)
7. [Technical Analysis & Discussion](#7-technical-analysis--discussion)
8. [Command Quick Reference Guide](#8-command-quick-reference-guide)
9. [Conclusion & Engineering Takeaways](#9-conclusion--engineering-takeaways)
10. [Repository Structure & Reproduction](#10-repository-structure--reproduction)

---

## 1. Objectives

1. **Infrastructure Provisioning**: Deploy Windows Subsystem for Linux (WSL 2) and Docker Desktop engine to support containerized Kubernetes nodes.
2. **Multi-Node Cluster Configuration**: Declare and provision a 3-node Kubernetes cluster consisting of one control-plane node (`kind-control-plane`) and two dedicated worker nodes (`kind-worker`, `kind-worker2`) using KinD v0.33 and YAML declarative manifests.
3. **Control Plane & Node Verification**: Verify the cluster control plane components (`kube-apiserver`, `etcd`, `kube-scheduler`, `kube-controller-manager`, and CNI networking) and inspect Docker container runtimes.
4. **Declarative Application Deployment**: Deploy a scalable NGINX web application across the cluster and analyze kube-scheduler pod distribution algorithms.
5. **Horizontal Scaling**: Scale the application workload dynamically from 3 replicas to 6 replicas, observing load distribution across worker nodes.
6. **Self-Healing Fault Tolerance Verification**: Test Kubernetes declarative state reconciliation by deliberately terminating active pods, observing ReplicaSet controller detection and automated zero-touch pod recreation.

---

## 2. Architectural Topology

Each Kubernetes node in KinD runs as an independent Docker container operating within the Windows Subsystem for Linux (WSL 2) virtualized environment. The control-plane node hosts core Kubernetes services, while worker nodes run the container runtime, `kubelet`, `kube-proxy`, and application Pods.

```mermaid
graph TD
    subgraph Host["Windows 11 Workstation (PowerShell CLI / kubectl / kind)"]
        subgraph DockerDesktop["Docker Desktop (WSL 2 Engine)"]
            subgraph ControlPlane["Node 1: kind-control-plane (Container: 6b16706fc603)"]
                API["API Server (Port 6443 / Host: 65242)"]
                ETCD["etcd Storage"]
                SCHED["kube-scheduler"]
                CM["kube-controller-manager"]
            end

            subgraph Worker1["Node 2: kind-worker (Container: 317ce8f4adcf)"]
                Kubelet1["kubelet + kube-proxy"]
                Pod1["Pod: web-5fc9f4bf66-hpncv<br/>(10.244.2.4)"]
                Pod2["Pod: web-5fc9f4bf66-lrvv7 / 49nq9<br/>(10.244.2.3)"]
                Pod3["Pod: web-5fc9f4bf66-w9rd5<br/>(10.244.2.2)"]
            end

            subgraph Worker2["Node 3: kind-worker2 (Container: fe5c2984d1b3)"]
                Kubelet2["kubelet + kube-proxy"]
                Pod4["Pod: web-5fc9f4bf66-9jr2v<br/>(10.244.1.3)"]
                Pod5["Pod: web-5fc9f4bf66-j5dv4<br/>(10.244.1.4)"]
                Pod6["Pod: web-5fc9f4bf66-rrnwx<br/>(10.244.1.2)"]
            end
        end
    end

    API --> Worker1
    API --> Worker2
    SCHED -.->|Schedules Pods| Worker1
    SCHED -.->|Schedules Pods| Worker2
```

---

## 3. Environment Specifications & Tools

| Component | Specification / Version | Role in Lab |
| :--- | :--- | :--- |
| **Operating System** | Windows 11 Enterprise / Pro (64-bit) | Host physical workstation OS |
| **CLI Environment** | Windows PowerShell 5.1 / PowerShell 7 (Admin) | Execution shell for cluster commands |
| **WSL Runtime** | Windows Subsystem for Linux (WSL 2, Ubuntu Distro) | Linux kernel compatibility layer for Docker engine |
| **Container Engine** | Docker Desktop (Engine v29.x, WSL 2 Backend) | Containerization host running node containers |
| **Kubernetes CLI** | `kubectl` v1.37.0 (Client) | Administrative CLI communicating with API server |
| **Cluster Tool** | `kind` (Kubernetes in Docker) v0.33.0 | Multi-node cluster orchestration engine |
| **Node Image** | `kindest/node:v1.37.0` | Base image containing Kubernetes node binaries |
| **Workload Image** | `nginx:latest` | Standard web server container workload |
| **Lab Directory** | `k8s-lab` | Working directory containing configuration files |

---

## 4. Cluster Configuration Manifest (`kind-config.yaml`)

To provision a production-like multi-node topology with separated control-plane and compute workers, the declarative manifest [`kind-config.yaml`](kind-config.yaml) was created:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
```

- **`kind: Cluster`**: Declares that KinD should instantiate a Kubernetes cluster.
- **`apiVersion: kind.x-k8s.io/v1alpha4`**: Uses KinD's v1alpha4 schema.
- **`nodes`**: Defines three distinct virtual node containers:
  - 1 control plane node hosting the API server, scheduler, and etcd database.
  - 2 worker nodes dedicated exclusively to scheduling and executing user workloads.

---

## 5. Step-by-Step Procedure & Execution Workflow

### Step 1: Environment Readiness & Tool Verification

Prior to cluster creation, the underlying WSL 2 virtualization layer and Docker runtime were verified:

```powershell
# Verify WSL 2 installation status
wsl --status

# List registered Linux distributions and WSL version
wsl -l -v
```

#### Observations:
- Default distribution: **Ubuntu**
- Default WSL version: **2**
- Active running distributions: `docker-desktop` (State: **Running**, Version: 2) and `Ubuntu` (Version: 2).

![WSL Status and Distribution Verification](images/k8s_wsl_status.png)
*Figure 1: PowerShell terminal confirming WSL 2 status and running Docker Desktop engine backend (`docker-desktop`, Version 2).*

---

### Step 2: Multi-Node Cluster Creation and Node Verification

The cluster was initiated using KinD and the custom configuration:

```powershell
# Navigate to lab workspace
cd k8s-lab

# Create 3-node Kubernetes cluster
kind create cluster --config kind-config.yaml

# Verify cluster connectivity
kubectl cluster-info --context kind-kind

# Verify node status
kubectl get nodes

# Inspect underlying Docker container instances
docker ps
```

#### Node Inventory & Status:
| Node Name | Kubernetes Status | Roles | Kubernetes Version | Underlying Docker Container ID | Mapped Ports |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **`kind-control-plane`** | **Ready** | `control-plane` | `v1.37.0` | `6b16706fc603` | `127.0.0.1:65242->6443/tcp` |
| **`kind-worker`** | **Ready** | `<none>` | `v1.37.0` | `317ce8f4adcf` | Internal CNI network |
| **`kind-worker2`** | **Ready** | `<none>` | `v1.37.0` | `fe5c2984d1b3` | Internal CNI network |

![Kubernetes Nodes and Docker Containers](images/k8s_nodes_docker_ps.png)
*Figure 2: Empirical output of `kubectl get nodes` verifying all 3 nodes in Ready status, and `docker ps` confirming each node executes inside an isolated Docker container on Windows.*

---

### Step 3: NGINX Deployment & Initial Pod Scheduling (3 Replicas)

A web server deployment was declared with an initial replica count of 3:

```powershell
# Deploy 3 replicas of the NGINX web server
kubectl create deployment web --image=nginx --replicas=3

# Verify deployment health
kubectl get deployments

# Inspect pod placement across worker nodes
kubectl get pods -o wide
```

#### Initial Pod Scheduling Distribution Table:
| Pod Name | Ready | Status | Restarts | Age | Pod IP Address | Node Assigned | Subnet CIDR |
| :--- | :---: | :---: | :---: | :---: | :---: | :--- | :--- |
| **`web-5fc9f4bf66-lrvv7`** | `1/1` | **Running** | 0 | 6m25s | `10.244.2.3` | **`kind-worker`** | `10.244.2.0/24` |
| **`web-5fc9f4bf66-w9rd5`** | `1/1` | **Running** | 0 | 6m25s | `10.244.2.2` | **`kind-worker`** | `10.244.2.0/24` |
| **`web-5fc9f4bf66-rrnwx`** | `1/1` | **Running** | 0 | 6m25s | **`kind-worker2`** | `10.244.1.0/24` |

![3-Replica NGINX Deployment Pod Placement](images/k8s_pods_3replicas.png)
*Figure 3: `kubectl get pods -o wide` displaying 3 NGINX replicas distributed across `kind-worker` (2 pods) and `kind-worker2` (1 pod) with distinct pod IPs.*

---

### Step 4: Horizontal Scaling to 6 Replicas

The deployment was scaled up by 100% (from 3 to 6 replicas) to observe how `kube-scheduler` handles increased load across the available worker compute pool:

```powershell
# Scale deployment from 3 to 6 replicas
kubectl scale deployment web --replicas=6

# Check deployment state
kubectl get deployments

# Inspect balanced pod placement
kubectl get pods -o wide
```

#### Scaled 6-Pod Scheduling & Load Balancing Matrix:
| Pod Name | Ready | Status | Restarts | Age | IP Address | Host Node | Node Workload Share |
| :--- | :---: | :---: | :---: | :---: | :---: | :--- | :---: |
| **`web-5fc9f4bf66-hpncv`** | `1/1` | **Running** | 0 | 50s | `10.244.2.4` | **`kind-worker`** | 3 / 6 (50%) |
| **`web-5fc9f4bf66-lrvv7`** | `1/1` | **Running** | 0 | 12m | `10.244.2.3` | **`kind-worker`** | 3 / 6 (50%) |
| **`web-5fc9f4bf66-w9rd5`** | `1/1` | **Running** | 0 | 12m | `10.244.2.2` | **`kind-worker`** | 3 / 6 (50%) |
| **`web-5fc9f4bf66-9jr2v`** | `1/1` | **Running** | 0 | 50s | `10.244.1.3` | **`kind-worker2`** | 3 / 6 (50%) |
| **`web-5fc9f4bf66-j5dv4`** | `1/1` | **Running** | 0 | 50s | `10.244.1.4` | **`kind-worker2`** | 3 / 6 (50%) |
| **`web-5fc9f4bf66-rrnwx`** | `1/1` | **Running** | 0 | 12m | `10.244.1.2` | **`kind-worker2`** | 3 / 6 (50%) |

![6-Replica Scaled Pod Distribution](images/k8s_pods_6replicas.png)
*Figure 4: `kubectl get pods -o wide` demonstrating 50/50 workload balancing across `kind-worker` (3 pods) and `kind-worker2` (3 pods).*

---

### Step 5: Fault Injection & Automated Self-Healing Verification

To test Kubernetes high availability and self-healing resilience, an active pod was forcefully deleted to simulate node or container failure:

```powershell
# Force delete an active pod on kind-worker
kubectl delete pod web-5fc9f4bf66-lrvv7

# Immediately check pod status to detect reconciliation
kubectl get pods
```

#### Observed Self-Healing Lifecycle:
1. **Target Pod Termination**: Pod `web-5fc9f4bf66-lrvv7` was targeted and deleted (`pod "web-5fc9f4bf66-lrvv7" deleted from default namespace`).
2. **State Discrepancy Detection**: The Kubernetes `Deployment` / `ReplicaSet` control loop immediately detected an actual count of 5 pods versus the declared desired count of 6 pods.
3. **Automated Replacement Scheduling**: Within **5 seconds**, the controller spawned a brand new replacement pod: **`web-5fc9f4bf66-49nq9`**.
4. **Transient Image Verification**: During the initial seconds, the replacement pod entered the image preparation phase (`0/1 ErrImagePull` / `ContainerCreating` transient state).

![Pod Deletion and Self-Healing Trigger](images/k8s_pod_delete_self_healing.png)
*Figure 5: PowerShell output showing the deliberate deletion of pod `web-5fc9f4bf66-lrvv7` and the instantaneous automated spawning of replacement pod `web-5fc9f4bf66-49nq9` by the ReplicaSet controller within 5 seconds.*

---

### Step 6: Steady-State Recovery & Final Pod Verification

Once the container image was pulled and initialized, the replacement pod `web-5fc9f4bf66-49nq9` transitioned smoothly to **`1/1 Running`**, maintaining the desired 6/6 availability across the cluster with zero manual intervention.

```powershell
# Confirm steady-state recovery of the replacement pod
kubectl get pods
```

#### Final Pod Status Table:
| Pod Name | Ready | Status | Restarts | Age | Lifecycle Role |
| :--- | :---: | :---: | :---: | :---: | :--- |
| **`web-5fc9f4bf66-49nq9`** | `1/1` | **Running** | 0 | 23m | **Automated Replacement (Self-Healed)** |
| **`web-5fc9f4bf66-9jr2v`** | `1/1` | **Running** | 0 | 25m | Stable Scaled Replica |
| **`web-5fc9f4bf66-hpncv`** | `1/1` | **Running** | 0 | 25m | Stable Scaled Replica |
| **`web-5fc9f4bf66-j5dv4`** | `1/1` | **Running** | 0 | 25m | Stable Scaled Replica |
| **`web-5fc9f4bf66-rrnwx`** | `1/1` | **Running** | 0 | 37m | Original Replica |
| **`web-5fc9f4bf66-w9rd5`** | `1/1` | **Running** | 0 | 37m | Original Replica |

![Self-Healing Completed and All 6 Pods Running](images/k8s_self_healing_healed.png)
*Figure 6: Final `kubectl get pods` output confirming that all 6 pods are in the `Running` state, with the self-healed replacement pod `web-5fc9f4bf66-49nq9` active and stable.*

---

## 6. Empirical Results & Verification

- **Cluster Health**: All 3 nodes (`kind-control-plane`, `kind-worker`, `kind-worker2`) transitioned to `Ready` status running Kubernetes `v1.37.0`.
- **Worker Isolation**: Workload pods were scheduled exclusively on worker nodes (`kind-worker` and `kind-worker2`), keeping the control plane isolated for core Kubernetes management components.
- **Dynamic Scaling**: Scaling from 3 to 6 replicas achieved exact 50% distribution per worker node (3 pods per node) without manual node assignment.
- **Failover & Recovery Time**: Pod termination detected immediately; replacement pod scheduled and initialized in **< 5 seconds**.

---

## 7. Technical Analysis & Discussion

### 1. Declarative Architecture vs. Imperative Management
In traditional virtualization and manual container environments, if an application process crashes or a VM halts, an administrator or external watchdog script must detect the failure and manually issue restart commands. 

In Kubernetes:
- The administrator declares the **Desired State** (`replicas: 6`).
- The **ReplicaSet Controller** continuously executes a reconciliation control loop:
  $$\text{Discrepancy} = \text{Desired Replicas} - \text{Actual Replicas}$$
- When pod `lrvv7` was terminated, the discrepancy became $6 - 5 = +1$.
- The controller immediately submitted an API request to schedule and instantiate replacement pod `49nq9`, driving the system back to parity.

### 2. Multi-Node Scheduling & Workload Spreading Mechanics
`kube-scheduler` assigns unscheduled pods to nodes using two filtering/scoring phases:
- **Filtering (Predicates)**: Filters out nodes that do not meet resource requirements, taints, or port constraints.
- **Scoring (Priorities)**: Scores remaining eligible candidate nodes. To minimize blast radius and ensure high availability, the default scoring algorithm favors spreading pods evenly across distinct failure domains (nodes).
- This is empirically demonstrated in Step 4: when scaling to 6 pods, exactly **3 pods were placed on `kind-worker`** and **3 pods were placed on `kind-worker2`**, achieving optimal $50\% - 50\%$ distribution without manual node selector rules.

### 3. Container Network Interface (CNI) IP Allocation
KinD provisions a Kindnet CNI bridge that provides unique, non-overlapping IPv4 subnets for each worker node:
- `kind-worker`: Assigned subnet `10.244.2.0/24` (Pods received IPs `10.244.2.2`, `10.244.2.3`, `10.244.2.4`).
- `kind-worker2`: Assigned subnet `10.244.1.0/24` (Pods received IPs `10.244.1.2`, `10.244.1.3`, `10.244.1.4`).

Every pod receives its own routable IP within the cluster SDN, enabling flat, direct pod-to-pod networking without NAT.

---

## 8. Command Quick Reference Guide

| Step | PowerShell Command | Functional Description |
| :---: | :--- | :--- |
| **1.1** | `wsl --status` / `wsl -l -v` | Inspect WSL 2 virtualization status and registered distributions. |
| **1.3** | `winget install -e --id Kubernetes.kubectl` | Install Kubernetes command-line management client. |
| **1.4** | `winget install -e --id Kubernetes.kind` | Install KinD (Kubernetes IN Docker) cluster provisioner. |
| **2.1** | `docker version` | Verify Docker client and daemon communication. |
| **3.1** | `mkdir k8s-lab; cd k8s-lab` | Create and navigate to dedicated lab workspace. |
| **3.4** | `notepad kind-config.yaml` | Write 3-node cluster declarative YAML manifest. |
| **4.1** | `kind create cluster --config kind-config.yaml` | Build the 3-node Kubernetes cluster. |
| **4.2** | `kubectl cluster-info --context kind-kind` | Verify API server and CoreDNS operational endpoints. |
| **4.3** | `kubectl get nodes` | List all cluster nodes and verify `Ready` state. |
| **4.4** | `docker ps` | View container runtimes backing the Kubernetes nodes. |
| **5.1** | `kubectl create deployment web --image=nginx --replicas=3` | Deploy 3 replicas of the NGINX web application. |
| **5.2** | `kubectl get deployments` | Inspect deployment status and ready replica count. |
| **5.3** | `kubectl get pods -o wide` | Display pod IP allocations and worker node placement. |
| **6.1** | `kubectl scale deployment web --replicas=6` | Horizontally scale NGINX deployment from 3 to 6 replicas. |
| **6.2** | `kubectl get pods -o wide` | Verify balanced multi-node distribution of 6 pods. |
| **6.3** | `kubectl delete pod <pod-name>` | Delete an active pod to trigger self-healing fault tolerance. |
| **6.4** | `kubectl get pods` | Verify automated replacement pod instantiation and healthy status. |

---

## 9. Conclusion & Engineering Takeaways

1. **Successful Multi-Node Deployment**: A production-grade multi-node Kubernetes cluster (1 control plane, 2 workers) was successfully instantiated on Windows using KinD and Docker Desktop with WSL 2.
2. **Automated Load Distribution**: The Kubernetes scheduler demonstrated intelligent, balanced pod distribution across worker nodes, scaling seamlessly from 3 replicas to 6 replicas with equal workload allocation across both worker nodes (3 pods per node).
3. **Resilient Self-Healing**: Deliberate deletion of an active production pod proved Kubernetes' robust self-healing capability. The `ReplicaSet` controller reconciled desired and actual states by spawning an automated replacement pod within 5 seconds, restoring cluster availability with zero human intervention.
4. **Cloud Infrastructure Mastery**: This experiment successfully reinforces core cloud computing concepts including declarative infrastructure as code (IaC), container orchestration, software-defined networking (CNI), pod scheduling, horizontal elasticity, and automated fault tolerance.

---

## 10. Repository Structure & Reproduction

### Directory Structure

```text
Experiment-4-Kubernetes-Cluster/
├── images/
│   ├── k8s_nodes_docker_ps.png
│   ├── k8s_pods_3replicas.png
│   ├── k8s_pods_6replicas.png
│   ├── k8s_pod_delete_self_healing.png
│   ├── k8s_self_healing_healed.png
│   └── k8s_wsl_status.png
├── kind-config.yaml
├── Experiment_4_Kubernetes_Lab_on_Windows_Manual.pdf
├── Experiment_4_Kubernetes_Cluster_Scheduling_and_Self_Healing_Report.docx
└── README.md
```

### Reproduction Steps

```powershell
# 1. Navigate to the experiment directory
cd Experiment-4-Kubernetes-Cluster

# 2. Provision the 3-node cluster using KinD
kind create cluster --config kind-config.yaml

# 3. Verify nodes are in Ready status
kubectl get nodes

# 4. Deploy 3 replicas of NGINX
kubectl create deployment web --image=nginx --replicas=3

# 5. Scale horizontally to 6 replicas
kubectl scale deployment web --replicas=6

# 6. Test self-healing by deleting a pod
kubectl get pods
kubectl delete pod <pod-name>
kubectl get pods

# 7. Clean up cluster resources when finished
kind delete cluster
```
