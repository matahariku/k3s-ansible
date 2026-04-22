# 🚀 K3s 3-Node Cluster Ansible Deployment

[![Cluster Status](https://img.shields.io/badge/Cluster-3%20Nodes%20Ready-brightgreen)](https://kubernetes.io/) 
[![K3s Version](https://img.shields.io/badge/K3s-v1.35.3%2Bk3s1-blue)](https://k3s.io/) 
[![Ansible](https://img.shields.io/badge/Ansible-Production%20Ready-purple)](https://ansible.com/)

Production-ready **3-node K3s cluster** deployment using Ansible. **Single command** to deploy 1 master + 2 dual-function workers.

## 🎯 **Cluster Overview**

| Node     | IP            | Role              | Status | Version        | Function          |
|----------|---------------|-------------------|--------|----------------|-------------------|
| **ops1** | `192.168.1.87`| `control-plane,etcd` | ✅ Ready | `v1.35.3+k3s1` | **Controlplane,etcd** |
| **dev1** | `192.168.1.88`| `worker`          | ✅ Ready | `v1.35.3+k3s1` | **Worker + Dev** |
| **ceph-0**| `192.168.1.86`| `worker`          | ✅ Ready | `v1.35.3+k3s1` | **Worker + Storage** |


## ✨ **Dual Function Workers**
- **`dev1`**: General workload + Development/CI/CD
- **`ceph-0`**: General workload + Ceph/Rook storage  

**Auto-labeled** `node-role.kubernetes.io/worker=` untuk NodeSelector/DaemonSet scheduling.

## 🚀 **Quick Start (1 Command)**

```bash
git clone https://github.com/matahariku/k3s-ansible
cd k3s-ansible
# Edit inventory.ini with your IP/host
ansible-playbook playbook.yaml
kubectl get nodes  # ✅ All Ready!
```

## �� **File Structure**
```bash
├── playbook.yaml # �� Production playbook (workers) 
├── inventory.ini # 🎯 Node configuration 
├── ansible.cfg # 🔇 Clean output (no warnings)
└── README.md # 📖 This file
```

## ⚙️ **Prerequisites**

```bash
# Required di semua nodes (ops1, dev1, ceph-0)
sudo apt update && sudo apt install -y curl iptables python3
# SSH key-based auth (fe user) ke semua nodes
ssh-copy-id fe@192.168.1.87 fe@192.168.1.88 fe@192.168.1.86
```

## 🔧 **Configuration**

**`inventory.ini`** (sesuaikan IP):
```ini
[k3s_workers]
dev1     ansible_host=192.168.1.88
ceph-0   ansible_host=192.168.1.86

[all:vars]
ansible_user=fe
ansible_ssh_private_key_file=~/.ssh/id_rsa
ansible_python_interpreter=/usr/bin/python3
```

**Controlplane ops1** (`192.168.1.87`) already have installed K3s server v1.35.3+k3s1.

## 🎪 **Deploy Sequence**

```bash
# 1. Deploy workers (otomatis cleanup + install + verify)
ansible-playbook playbook.yaml

# 2. Verify cluster
kubectl get nodes -o wide
```

**Output expected:**
```bash
NAME STATUS ROLES AGE VERSION
ceph-0 Ready worker 2m v1.35.3+k3s1
dev1 Ready worker 2m v1.35.3+k3s1
ops1 Ready control-plane,etcd 24h v1.35.3+k3s1
```

## 🧪 **Test Cluster**

```bash
# Test pod scheduling to all nodes
kubectl run nginx --image=nginx --replicas=3 --restart=Never
kubectl get pods -o wide

# Deploy ke worker only
kubectl run dev-app --image=nginx --overrides='{"spec":{"nodeSelector":{"node-role.kubernetes.io/worker":""}}}'
```

## 🔄 **Maintenance**

```bash
# Re-run (idempotent - safe)
ansible-playbook playbook.yaml

# Scale up (tambah worker)
echo "new-worker ansible_host=192.168.1.99" >> inventory.ini
ansible-playbook playbook.yaml
```

## 🎨 **Future Plans**

```bash
kubectl get nodes  # cluster production is ready!
```
## 📖 **Journey to HA Dual Cluster**
```bash
🚀 Phase 1: Single Master (ops1)
→ playbook.yaml → 1 control-plane + 2 agents

⚠️  Phase 2: NotReady nodes (version mismatch)
→ Fixed: v1.35.3+k3s1 + cleanup + token fix

✅ Phase 3: Production Single Master ✓
→ dev1 + ceph-0 = Ready workers

🎯 Phase 4: FULL HA etcd (3 servers) ← NOW!
→ cleanup-ha.yaml + playbook-ha.yaml 
→ 3x control-plane,etcd + worker labels

💚 RESULT: HA Dual Function Cluster LIVE!
```

## 🎯 **Final Cluster Architecture**

| Node   | IP                                               | Roles        | Status              | Version              | Function |
| ------ | ------------------------------------------------ | ------------ | ------------------- | -------------------- | -------- |
| ops1   | 192.168.1.87\| control-plane,etcd                | ✅ Ready     | v1.35.3+k3s1        | Master               |          |
| dev1   | 192.168.1.88\| control-plane,etcd,worker         | ✅ Ready     | v1.35.3+k3s1        | Dev + Workloads      |          |          
| ceph-0 | 192.168.1.86\| control-plane,etcd,worker         | ✅ Ready     | v1.35.3+k3s1         | Storage + Workloads |          |          

## 💚 **ALL 3 NODES LIVE - 28m uptime - HA etcd quorum!**
## ✨ **Production Features**
```bash
✅ 3x control-plane → API server HA
✅ 3x embedded etcd → Quorum (tollerance 1 node down) 
✅ Dual function → all nodes can workloads
✅ Auto snapshots → etcd backup setiap 6 jam
✅ Worker labels → NodeSelector ready
✅ Idempotent → ansible-playbook in secure repeatiton
```

## 📋 **File Structure (Production) - FINAL**
```bash
├── playbook.yaml        # Single master (legacy)  
├── playbook-ha.yaml     # 🔥 HA etcd (current)
├── inventory.ini        # Single master (ops1 + workers)
├── inventory-ha.ini     # 🔥 HA 3-server (semua control-plane)
├── cleanup-ha.yaml      # Full reset HA
├── ansible.cfg          # Clean output
└── README.md            # This epic journey
```
## 🎪 **Usage Guide (2 Scenarios)**

```bash
# Test connectivity before start playbook
ansible -i inventory.ini all -m ping

# SINGLE MASTER (inventory.ini)
ansible-playbook -i inventory.ini playbook.yaml
→ ops1 (master) + dev1/ceph-0 (workers)

# Test connectivity before start playbook
ansible -i inventory-ha.ini all -m ping

# FULL HA (inventory-ha.ini) ← RECOMMENDED
ansible-playbook -i inventory-ha.ini cleanup-ha.yaml
ansible-playbook -i inventory-ha.ini playbook-ha.yaml
→ 3x control-plane,etcd (ops1+dev1+ceph-0)
```
## **get information nodes**
```bash
# Fix kubect
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown fe:fe ~/.kube/config
export KUBECONFIG=~/.kube/config

# Get all nodes
k get nodes -o wide
```

## **Label dev1 ceph-0 tobe Worker**
```bash
kubectl label node dev1 ceph-0 \
  node-role.kubernetes.io/worker= \
  workload=production \
  tier=worker
node/dev1 labeled
node/ceph-0 labeled
```

## 🔍 **HA Verification**
```bash
# etcd quorum (3 members)
sudo k3s etcd-snapshot list

# Roles + labels
k get nodes -o custom-columns=NAME:.metadata.name,ROLES:.metadata.labels.'node-role\.kubernetes\.io/master',WORKER:.metadata.labels.'node-role\.kubernetes\.io/worker'

# No taints = dual function
k describe node ops1 | grep Taints  # <none>
```
## 🎨 **Future Plans**

```yaml
# TODO: MetalLB LoadBalancer
# TODO: Rook Ceph Storage (ceph-0)  
# TODO: ArgoCD GitOps
# TODO: Prometheus + Grafana
# TODO: Longhorn Backup
```

## 🛡️ **Production Features**

- ✅ **Idempotent** - Safe to run multiple times
- ✅ **Version pinned** - v1.35.3+k3s1 exact match  
- ✅ **Auto-cleanup** - Remove old installations
- ✅ **Network verify** - Test API connectivity
- ✅ **Production-ready** - Error handling + retries
- ✅ **3x etcd quorum** (tolleransi 1 node down)
- ✅ **Dual function** (controlplane + workloads)
- ✅ **Auto etcd snapshots** (6 hours)
- ✅ **Worker labels** (NodeSelector ready)
- ✅ **TLS SAN multi-IP**
- ✅ **Idempotent playbooks**
 
