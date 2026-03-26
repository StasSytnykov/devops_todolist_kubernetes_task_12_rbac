# Validate RBAC

**Prereqs:** [kind](https://kind.sigs.k8s.io/), `kubectl`, repo cloned.

1. **Cluster:** `kind create cluster --config cluster.yml`

2. **Apply:** Put `.infrastructure/app/ns.yml` and `.infrastructure/security/rbac.yml` on the cluster before the Deployment, then apply the rest of your manifests (ConfigMap, secrets, PVC, DB, app, etc.) until `todoapp` pods are Running.

3. **Check ServiceAccount on Deployment:**  
   `kubectl get deploy todoapp -n todoapp -o jsonpath='{.spec.template.spec.serviceAccountName}{"\n"}'` → should print `secrets-reader`.

4. **Call the API from the pod** (uses the mounted ServiceAccount token). The Role allows **`list`** and **`get`** on `secrets` in `todoapp`.

```bash
POD=$(kubectl get pods -n todoapp -l app=todoapp -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n todoapp "$POD" -- sh -c '
SERVICEACCOUNT=/var/run/secrets/kubernetes.io/serviceaccount
APISERVER=https://kubernetes.default.svc
TOKEN=$(cat ${SERVICEACCOUNT}/token)
CACERT=${SERVICEACCOUNT}/ca.crt

# List secrets (needs verb: list)
curl --cacert "$CACERT" -H "Authorization: Bearer ${TOKEN}" \
  "${APISERVER}/api/v1/namespaces/todoapp/secrets"
'
```

- **List:** expect `"kind": "SecretList"`.  
- **Get:** expect `"kind": "Secret"` with a `"data"` map (base64).  
- **`403 Forbidden`:** Role/RoleBinding or verbs are wrong for that URL.

**Cleanup:** `kind delete cluster`
