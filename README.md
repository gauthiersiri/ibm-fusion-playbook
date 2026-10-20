# IBM Storage Fusion — Ansible Playbook

Modular and idempotent Ansible playbook to deploy **IBM Storage Fusion**, **Fusion Data Foundation (FDF)**, **MetalLB**, **OpenShift Virtualization**, and **MultiCluster Engine (MCE)** on OpenShift.

---

## 🚀 Quick Start

### 1. Requirements & Setup
```bash
source ~/ansible-venv/bin/activate
ansible-galaxy collection install -r requirements.yml
```

### 2. Configuration
Copy the sample file and set your cluster credentials:
```bash
cp inventory/host_vars/ocp_cluster.yml.example inventory/host_vars/ocp_cluster.yml
```

### 3. Run
```bash
ansible-playbook site.yml -i inventory/hosts.yml
```

---

## 🎛️ Component Selection

Enable or disable components in `inventory/host_vars/ocp_cluster.yml` or via CLI extra vars (`-e`):

| Flag | Default | Description |
|---|---|---|
| `deploy_prerequisites` | `true` | Global pull secret (`cp.icr.io`) + IBM CatalogSource |
| `deploy_fusion_operator` | `true` | Fusion Operator (`isf-operator`) + License |
| `deploy_fusion_data_foundation` | `true` | Fusion Data Foundation (Ceph / Local Storage) |
| `deploy_metallb` | `false` | MetalLB (*auto-enabled in provider mode or with OpenShift Virtualization*) |
| `deploy_hcp_lvm_config` | `false` | LVM ConfigMap for HCP (*auto-enabled when MCE is enabled*) |
| `deploy_fusion_backup` | `false` | Fusion Backup & Restore + MinIO mirror |
| `deploy_openshift_virtualization` | `false` | OpenShift Virtualization (CNV / KubeVirt) |
| `deploy_multicluster_engine` | `false` | MultiCluster Engine (MCE / Hosted Control Planes) |

---

## 🔗 Automated Dependencies

The playbook enforces strict deployment ordering and dependencies:
1. **MetalLB** is auto-deployed when `fdf_deployment_mode: provider` or `deploy_openshift_virtualization: true`.
2. **`hcp_lvm_config`** runs **before** Data Foundation whenever `deploy_multicluster_engine: true`.
3. **Backup & Restore** runs **after** Data Foundation.
4. **MCE** provisions KubeVirt credentials automatically if virtualization is present.

---

## 💡 Usage Examples

```bash
# 1. Full deployment (as configured in host_vars)
ansible-playbook site.yml -i inventory/hosts.yml

# 2. Minimal base only (skip Data Foundation, Backup, Virtualization, MCE)
ansible-playbook site.yml -i inventory/hosts.yml \
  -e deploy_fusion_data_foundation=false

# 3. FDF in Provider mode (automatically deploys MetalLB)
ansible-playbook site.yml -i inventory/hosts.yml \
  -e fdf_deployment_mode=provider

# 4. Virtualization & MultiCluster Engine stack
ansible-playbook site.yml -i inventory/hosts.yml \
  -e deploy_openshift_virtualization=true \
  -e deploy_multicluster_engine=true
```

---

## 📁 Project Structure

```text
.
├── site.yml                             # Main playbook orchestrating roles
├── group_vars/openshift_clusters.yml    # Global defaults
├── inventory/
│   ├── hosts.yml                        # Inventory
│   └── host_vars/ocp_cluster.yml        # Target cluster variables (gitignored)
└── roles/
    ├── auth/                            # OpenShift authentication (_k8s_auth)
    ├── prerequisites/                   # Global pull secret & IBM CatalogSource
    ├── fusion_operator/                 # Fusion Operator & License
    ├── metallb/                         # MetalLB (Operator + Pool + L2Adv)
    ├── hcp_lvm_config/                  # LVM ConfigMap for Hosted Control Planes
    ├── fusion_data_foundation/          # FDF / Ceph (Local or Provider mode)
    ├── fusion_backup/                   # Backup & Restore + MinIO mirror
    ├── openshift_virtualization/        # OpenShift Virtualization (CNV)
    └── multicluster_engine/             # MultiCluster Engine (MCE)
```
