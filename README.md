# IBM Storage Fusion — Ansible Playbook

Playbook Ansible pour installer et configurer **IBM Storage Fusion** sur un cluster OpenShift.

## Prérequis

| Outil | Version minimale |
|---|---|
| Ansible | 2.14+ |
| `kubernetes.core` collection | 2.4+ |
| `oc` CLI | dans le `PATH` |

```bash
# Activer le venv
source ~/ansible-venv/bin/activate

# Installer les collections
ansible-galaxy collection install -r requirements.yml
```

## Configuration

Toutes les variables sont dans **`inventory/host_vars/ocp_cluster.yml`** — pas de `-e` sur la ligne de commande.

Copier le fichier d'exemple et remplir les valeurs :

```bash
cp inventory/host_vars/ocp_cluster.yml.example inventory/host_vars/ocp_cluster.yml
# puis éditer le fichier
```

### Variables obligatoires

| Variable | Description |
|---|---|
| `ocp_api_url` | API endpoint du cluster (`https://api.<cluster>.<domain>:6443`) |
| `ocp_username` + `ocp_password` | Authentification par identifiant (option A) |
| `ocp_kubeconfig` | Chemin vers le kubeconfig (option B) |
| `ocp_entitlement_key` | Clé IBM depuis [myibm.ibm.com](https://myibm.ibm.com/products-services/containerlibrary) |
| `fusion_version` | Version Fusion à installer (ex : `2.14.0`, `2.12.4`) |

### Variables optionnelles importantes

| Variable | Défaut | Description |
|---|---|---|
| `ocp_validate_certs` | `true` | Mettre `false` pour les clusters avec certificat auto-signé |
| `odf_storage_nodes` | `[]` | Liste explicite des nœuds ODF (vide = auto-détection) |
| `odf_node_selector_key` | `node-role.kubernetes.io/worker` | Label pour l'auto-détection des nœuds ODF |
| `odf_disk_min_size` | `""` | Taille min des disques pour `LocalVolumeSet` (ex : `490Gi`) |
| `odf_disk_max_size` | `""` | Taille max des disques pour `LocalVolumeSet` (ex : `510Gi`) |

Les defaults complets sont dans [`group_vars/openshift_clusters.yml`](group_vars/openshift_clusters.yml).

## Utilisation

```bash
# Run complet (auth + prérequis + opérateur + ODF)
ansible-playbook site.yml

# Seulement les prérequis (pull secret, catalog source)
ansible-playbook site.yml --tags prerequisites

# Seulement l'opérateur Fusion
ansible-playbook site.yml --tags fusion_operator

# Seulement Data Foundation (ODF)
ansible-playbook site.yml --tags fusion_data_foundation

# Dry-run — attention : les étapes "create + wait" échouent en --check
ansible-playbook site.yml --check
```

## Structure du projet

```
.
├── ansible.cfg                          # Configuration Ansible
├── site.yml                             # Playbook principal
├── requirements.yml                     # Collections requises
│
├── inventory/
│   ├── hosts.yml                        # Inventaire (groupe openshift_clusters)
│   └── host_vars/
│       └── ocp_cluster.yml              # Variables du cluster (gitignored)
│
├── group_vars/
│   └── openshift_clusters.yml           # Valeurs par défaut
│
└── roles/
    ├── auth/                            # Authentification OCP → _k8s_auth
    ├── prerequisites/                   # Pull secret + CatalogSource
    ├── fusion_operator/                 # Installation opérateur + licence
    └── fusion_data_foundation/          # ODF en mode local (LSO + ODF)
```

## Flux d'exécution

```
site.yml
├── [always] role: auth
│     └─ Détection auto cert/token → produit _k8s_auth
│
├── [prerequisites] role: prerequisites
│     ├─ pull_secret.yml   → merge cp.icr.io dans openshift-config/pull-secret
│     ├─ catalog_source.yml→ ibm-operator-catalog (check-before-create, wait READY)
│     └─ verify.yml        → vérification CatalogSource
│
├── [fusion_operator] role: fusion_operator
│     ├─ namespace.yml     → ibm-spectrum-fusion-ns + OperatorGroup
│     ├─ subscription.yml  → Subscription + approbation InstallPlan (filtre ownerRef)
│     ├─ wait.yml          → CSV Succeeded + 11 deployments ready
│     └─ license.yml       → SpectrumFusion CR + attente 5 deployments post-licence
│
└── [fusion_data_foundation] role: fusion_data_foundation
      ├─ discover_nodes.yml → liste explicite ou auto-détection (≥ 3 nœuds)
      ├─ label_nodes.yml    → cluster.ocs.openshift.io/openshift-storage=""
      ├─ fsi.yml            → FusionServiceInstance odfmanager (backingStorageType: Local)
      ├─ wait_lso.yml       → openshift-local-storage namespace + CSV Succeeded
      ├─ local_volumes.yml  → LocalVolumeDiscovery + LocalVolumeSet + attente PVs
      └─ wait_odf.yml       → progressPercentage == 100
```

## Versions supportées

| Branche | Versions |
|---|---|
| 2.14.x | `2.14.0` |
| 2.13.x | `2.13.0` · `2.13.1` · `2.13.2` |
| 2.12.x | `2.12.0` · `2.12.1` · `2.12.2` · `2.12.3` · `2.12.4` |
| 2.11.x | `2.11.0` · `2.11.1` · `2.11.2` |

Channel OLM : toujours **`v2.0`** pour toute la génération 2.x.

## Références

- [IBM Fusion 2.14 — Déploiement CLI](https://www.ibm.com/docs/en/fusion-software/2.14.0?topic=fusion-deploying-using-cli)
- [IBM Fusion — Data Foundation local mode](https://www.ibm.com/docs/en/fusion-software/2.11.0?topic=foundation-configuring-fusion-data-in-local-mode)
- [IBM Entitlement Key](https://myibm.ibm.com/products-services/containerlibrary)
- [kubernetes.core collection](https://docs.ansible.com/ansible/latest/collections/kubernetes/core/)
