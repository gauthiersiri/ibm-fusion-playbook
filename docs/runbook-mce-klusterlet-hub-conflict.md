# Runbook — Conflit de klusterlet MCE/ACM (hub supérieur écrase local-cluster)

## Contexte

Un cluster OpenShift héberge MCE (Multicluster Engine). Ce MCE est lui-même importé
dans un hub ACM de niveau supérieur (appelé **hub01** ici). Il n'existe qu'un seul
klusterlet par cluster. L'import dans hub01 écrase le klusterlet créé par le MCE local
pour `local-cluster`, ce qui empêche le MCE de fonctionner correctement.

**Variables à adapter à chaque environnement :**

| Variable | Exemple session | Description |
|---|---|---|
| `<CLUSTER_API>` | `https://api.itz-ypo8pn.hub04-lb.techzone.ibm.com:6443` | API du cluster cible |
| `<HUB_API>` | `https://api.hub04.osv.techzone.ibm.com:6443` | API du hub supérieur |
| `<HUB_CLUSTER_NAME>` | `itz-ypo8pn` | Nom du cluster dans le hub supérieur (`klusterlet.spec.clusterName`) |

---

## 1. Diagnostic

### 1.1 Symptômes observables

- `local-cluster` en `Available=Unknown`, raison `ManagedClusterLeaseUpdateStopped`
- Tous les addons (`hypershift-addon`, `cluster-proxy`, `work-manager`, `managed-serviceaccount`) en `Unknown` / `unreachable`
- La console MCE refuse la création d'un cluster HCP : *"Hosted control plane operator must be enabled"*

### 1.2 Commandes de diagnostic

```bash
# 1. Vérifier le clusterName du klusterlet — doit être "local-cluster", pas le nom dans hub01
oc get klusterlet klusterlet -o jsonpath='{.spec.clusterName}{"\n"}'
# → Si retourne <HUB_CLUSTER_NAME> au lieu de "local-cluster" : problème confirmé

# 2. Vérifier vers quel hub pointe le bootstrap-hub-kubeconfig
oc get secret bootstrap-hub-kubeconfig -n open-cluster-management-agent \
  -o jsonpath='{.data.kubeconfig}' | base64 -d | grep "server:"
# → Si retourne <HUB_API> : problème confirmé

# 3. Vérifier le lease (date du dernier renouvellement)
oc get lease managed-cluster-lease -n local-cluster \
  -o jsonpath='{.spec.renewTime}{"\n"}'
# → Si la date est ancienne et ne progresse pas : l'agent ne renouvelle plus

# 4. Vérifier les conditions du ManagedCluster
oc get managedcluster local-cluster -o jsonpath=\
'{range .status.conditions[*]}{.type}{"\t"}{.status}{"\t"}{.reason}{"\n"}{end}'
```

### 1.3 Cause racine

Il n'existe qu'**un seul klusterlet par cluster**. Quand hub01 importe le cluster sous
le nom `<HUB_CLUSTER_NAME>`, il déploie via ManifestWork un klusterlet avec
`spec.clusterName: <HUB_CLUSTER_NAME>` et un `bootstrap-hub-kubeconfig` pointant
vers `<HUB_API>`. Cela écrase le klusterlet du MCE local (`clusterName: local-cluster`).
L'agent envoie ses leases à hub01, plus au MCE interne — `local-cluster` passe
`Available=Unknown` après 5 minutes de grâce.

**Preuves complémentaires :**
```bash
# Identifier les AppliedManifestWorks déposés par hub01 (klusterlet + crds)
oc get appliedmanifestworks \
  -l import.open-cluster-management.io/klusterlet-works=true
# → Deux entrées avec le hash de hub01 et le nom <HUB_CLUSTER_NAME>-klusterlet(/-crds)
# → Peuvent être absents si le work-agent de hub01 a déjà nettoyé
```

---

## 2. Résolution

> **Ordre impératif :** supprimer le klusterlet *avant* de poser les VAPs.
> Le work-agent de hub01 participe au cleanup du finalizer — si les VAPs sont
> posées avant, elles bloquent les UPDATE du work-agent pendant le cleanup et
> le finalizer ne se lève jamais.

### Étape 1 — Supprimer le klusterlet

```bash
oc delete klusterlet klusterlet
```

Attendre que le klusterlet disparaisse et que le namespace se vide :
```bash
watch oc get klusterlet,pods -n open-cluster-management-agent
# → "No resources found" sur les deux
```

Le finalizer se lève normalement grâce au work-agent de hub01 encore actif.
**Ne pas forcer le patch du finalizer** — si le `delete` prend plus de 2 minutes,
vérifier les logs du pod `klusterlet` dans `open-cluster-management-agent`.

### Étape 2 — Poser les ValidatingAdmissionPolicies

```bash
cat <<'EOF' | oc apply -f -
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: deny-hub01-klusterlet
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
    - apiGroups: ["operator.open-cluster-management.io"]
      apiVersions: ["v1"]
      operations: ["CREATE", "UPDATE"]
      resources: ["klusterlets"]
  validations:
  - expression: "object.spec.clusterName != '<HUB_CLUSTER_NAME>'"
    message: "Le clusterName <HUB_CLUSTER_NAME> est reserve au MCE local. Modification refusee."
    reason: Forbidden
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: deny-hub01-klusterlet
spec:
  policyName: deny-hub01-klusterlet
  validationActions: ["Deny"]
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: deny-hub01-bootstrap-kubeconfig
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
    - apiGroups: [""]
      apiVersions: ["v1"]
      operations: ["CREATE", "UPDATE"]
      resources: ["secrets"]
    namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: open-cluster-management-agent
  validations:
  - expression: >
      object.metadata.name != 'bootstrap-hub-kubeconfig' ||
      request.userInfo.username.startsWith('system:serviceaccount:open-cluster-management') ||
      request.userInfo.groups.exists(g, g == 'system:cluster-admins' || g == 'cluster-admins')
    message: "Modification de bootstrap-hub-kubeconfig refusee - seul le MCE local ou un admin peut ecrire ce secret"
    reason: Forbidden
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: deny-hub01-bootstrap-kubeconfig
spec:
  policyName: deny-hub01-bootstrap-kubeconfig
  validationActions: ["Deny"]
EOF
```

**Validation de la VAP klusterlet :**
```bash
oc get klusterlet klusterlet -o json \
  | jq '.spec.clusterName="<HUB_CLUSTER_NAME>" | del(.metadata.resourceVersion,.metadata.uid,.status)' \
  | oc apply --dry-run=server -f -
# → Doit retourner : "denied request: Le clusterName <HUB_CLUSTER_NAME> est reserve..."
```

### Étape 3 — Réappliquer l'import MCE local

```bash
# CRDs d'abord
oc get secret local-cluster-import -n local-cluster \
  -o jsonpath='{.data.crds\.yaml}' | base64 -d | oc apply -f -

# Puis le manifest complet
oc get secret local-cluster-import -n local-cluster \
  -o jsonpath='{.data.import\.yaml}' | base64 -d | oc apply -f -
```

**Vérification immédiate :**
```bash
oc get klusterlet klusterlet -o jsonpath='{.spec.clusterName}{"\n"}'
# → "local-cluster" ✓

oc get secret bootstrap-hub-kubeconfig -n open-cluster-management-agent \
  -o jsonpath='{.data.kubeconfig}' | base64 -d | grep "server:"
# → "https://kubernetes.default.svc:443" ✓
```

### Étape 4 — Supprimer les AppliedManifestWorks de hub01 (si présents)

```bash
HUB_HASH=$(oc get appliedmanifestwork \
  -l import.open-cluster-management.io/klusterlet-works=true \
  -o jsonpath='{.items[0].spec.hubHash}' 2>/dev/null)

if [ -n "$HUB_HASH" ]; then
  oc delete appliedmanifestwork \
    ${HUB_HASH}-<HUB_CLUSTER_NAME>-klusterlet \
    ${HUB_HASH}-<HUB_CLUSTER_NAME>-klusterlet-crds \
    --ignore-not-found
  echo "AppliedManifestWorks supprimes"
else
  echo "Aucun AppliedManifestWork a supprimer"
fi
```

---

## 3. Validation finale

```bash
# 1. clusterName du klusterlet
oc get klusterlet klusterlet -o jsonpath='{.spec.clusterName}{"\n"}'
# → "local-cluster" ✓

# 2. Connexion hub du klusterlet
oc get klusterlet klusterlet -o jsonpath=\
'{.status.conditions[?(@.type=="HubConnectionDegraded")].message}{"\n"}'
# → "...to apiserver https://kubernetes.default.svc:443 is working" ✓

# 3. ManagedCluster disponible
oc get managedcluster local-cluster \
  -o jsonpath='{.status.conditions[?(@.type=="ManagedClusterConditionAvailable")].status}{"\n"}'
# → "True" ✓

# 4. Lease renouvelé (comparer 2 appels à ~1 min d'intervalle)
oc get lease managed-cluster-lease -n local-cluster \
  -o jsonpath='{.spec.renewTime}{"\n"}'
# → La date doit progresser ✓

# 5. Addons disponibles
oc get managedclusteraddons -n local-cluster
# → hypershift-addon, cluster-proxy, work-manager, managed-serviceaccount : AVAILABLE=True ✓
# → hypershift-addon peut prendre ~2 min de plus que les autres

# 6. VAPs en place
oc get validatingadmissionpolicy,validatingadmissionpolicybinding | grep deny-hub01
# → 2 policies + 2 bindings ✓
```

---

## 4. Problèmes rencontrés (sessions de validation)

### P1 — Finalizer bloqué (artefact VAP posée trop tôt)
**Symptôme :** `oc delete klusterlet` posait un `deletionTimestamp` mais le klusterlet
restait présent indéfiniment.
**Cause :** Les VAPs étaient posées *avant* le delete. Le work-agent de hub01 tentait
de mettre à jour le klusterlet pendant son cleanup (opération `UPDATE`) — la VAP
bloquait cette opération, le finalizer ne se levait jamais.
**Résolution :** Supprimer le klusterlet *avant* de poser les VAPs. Le finalizer se
lève naturellement en ~10 secondes.

### P2 — VAP bootstrap bloque le réimport par kubeadmin
**Symptôme :** `oc apply` du `import.yaml` échoue avec `secrets "bootstrap-hub-kubeconfig" is forbidden`.
**Cause :** La VAP n'autorisait que `system:serviceaccount:open-cluster-management*`
mais l'apply est fait par `kubeadmin` qui appartient au groupe `system:cluster-admins`.
**Résolution :** Ajouter l'exception sur les groupes `system:cluster-admins` et
`cluster-admins` dans l'expression CEL de la VAP.

### P3 — bootstrap-hub-kubeconfig réécrit par hub01 après l'apply
**Symptôme :** Logs de `klusterlet-agent` avec
`User "system:serviceaccount:<HUB_CLUSTER_NAME>:...-bootstrap-sa" cannot list ...`.
Le lease ne se renouvelle plus après le premier tick.
**Cause :** Le work-agent de hub01 était encore actif et a réécrit le secret juste
après l'apply (observé sur les deux premiers clusters, pas le troisième).
**Résolution :** Supprimer les deux secrets et réappliquer, puis redémarrer l'agent.
Avec la VAP bootstrap en place, ce cas ne peut plus se produire.

### P4 — Addons restent Unknown quelques minutes
**Symptôme :** Pods Running mais `AVAILABLE=Unknown` persistant.
**Cause :** Délai normal de renouvellement de lease au démarrage.
**Résolution :** Attendre 2-3 minutes. `hypershift-addon` est généralement le dernier.

---

## 5. Risque résiduel

Si hub01 **conserve un credential admin** (token OAuth, kubeconfig avec `cluster-admin`)
sur l'API de ce cluster, il peut recréer un ManagedCluster `<HUB_CLUSTER_NAME>` avec
`auto-import-secret` et relancer le cycle entier. La VAP sur le klusterlet bloquera
la réécriture du klusterlet, mais hub01 pourrait tenter d'autres actions.

**Signe d'un réimport par hub01 :**
```bash
oc get klusterlet klusterlet -o jsonpath='{.spec.clusterName}{"\n"}'
# → Retourne <HUB_CLUSTER_NAME> au lieu de "local-cluster"
```

**Solution propre (nécessite accès à hub01) :**
Demander à l'admin de hub01 de supprimer le ManagedCluster `<HUB_CLUSTER_NAME>` et de
confirmer l'absence d'`auto-import-secret` dans le namespace correspondant.

---

## 6. Pièges à éviter

| Piège | Explication |
|---|---|
| Poser les VAPs avant le `oc delete klusterlet` | Le work-agent de hub01 ne peut plus participer au cleanup → finalizer bloqué indéfiniment |
| Ne pas ajouter l'exception `cluster-admins` sur la VAP bootstrap | `oc apply` par kubeadmin est bloqué — impossible de réimporter |
| Filtrer la VAP klusterlet sur l'identité (certificat, groupe OCM) | Ces identités sont émises par le CA de hub01 et ne sont pas reconnues par l'API locale |
| Bloquer sur l'identité du work-agent | C'est le même ServiceAccount utilisé par le MCE local — se bloquer soi-même |
| Appliquer `DELETE` dans la VAP klusterlet | `object` est `null` sur DELETE en CEL — la validation serait inopérante |
| Forcer le patch du finalizer sans raison | Jamais nécessaire si l'ordre est respecté |
