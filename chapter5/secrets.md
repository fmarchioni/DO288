# Come cifrare i Secrets usando il Secrets Store CSI Driver Operator

# ✅ **1. Installare il Secrets Store CSI Driver Operator**

Installa l’operatore nel namespace `openshift-operators`:

```bash
oc apply -f - <<'EOF'
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: secrets-store-csi-driver
  namespace: openshift-operators
spec:
  channel: stable
  installPlanApproval: Automatic
  name: secrets-store-csi-driver-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```

Verifica che l’operatore sia installato:

```bash
oc get csv -n openshift-operators | grep csi
```

---

# 🔐 **2. Preparare HashiCorp Vault**

Questa parte è fatta nel lato Vault (non su OpenShift).
Ipotizziamo:

* Vault è raggiungibile all’URL:
  `https://vault.example.com:8200`
* Path dei segreti: `secret/data/myapp/db`
* Secret keys:

  * `username`
  * `password`

### Crea il secret in Vault

```bash
vault kv put secret/myapp/db username="myuser" password="mypassword"
```

---

# 🔐 **3. Configurare il metodo di autenticazione Kubernetes in Vault**

Serve perché Vault identifichi i Pod tramite il loro ServiceAccount.

### a) Abilita Kubernetes auth

```bash
vault auth enable kubernetes
```

### b) Configura il backend Kubernetes di Vault

Estrarre il token JWT del SA di default (solo per configurazione):

```bash
SA_SECRET=$(oc get sa default -o jsonpath='{.secrets[0].name}')
TOKEN=$(oc get secret $SA_SECRET -o jsonpath='{.data.token}' | base64 -d)
```

Estrai anche l’endpoint API del cluster:

```bash
K8S_HOST=$(oc whoami --show-server | sed 's/https:\/\///')
```

Configura Vault:

```bash
vault write auth/kubernetes/config \
  token_reviewer_jwt="$TOKEN" \
  kubernetes_host="https://$K8S_HOST" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
```

### c) Crea la role di Vault che mappa SA → policy

```bash
vault write auth/kubernetes/role/myapp-role \
  bound_service_account_names=myapp-sa \
  bound_service_account_namespaces=myproject \
  policies=myapp-policy \
  ttl=1h
```

### d) Crea la policy in Vault per leggere i secret

```bash
vault policy write myapp-policy - <<EOF
path "secret/data/myapp/db" {
  capabilities = ["read"]
}
EOF
```

---

# 🔐 **4. Preparare OpenShift: ServiceAccount dedicato**

Nel namespace del workload (es. `myproject`):

```bash
oc new-project myproject
```

```bash
oc apply -f - <<'EOF'
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp-sa
EOF
```

---

# 🔐 **5. Creare il SecretProviderClass (SPC)**

Questo è il cuore dell’integrazione.

```bash
oc apply -f - <<'EOF'
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: vault-database-creds
  namespace: myproject
spec:
  provider: vault
  parameters:
    vaultAddress: "https://vault.example.com:8200"
    roleName: "myapp-role"
    objects: |
      - objectName: "db-user"
        secretPath: "secret/data/myapp/db"
        secretKey: "username"
      - objectName: "db-pass"
        secretPath: "secret/data/myapp/db"
        secretKey: "password"
EOF
```

Questo **non crea Secret in OpenShift**: i valori vengono letti dinamicamente da Vault.

---

# 📦 **6. Modificare il Deployment per montare i secrets**

Esempio Deployment completo:

```bash
oc apply -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: myproject
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      serviceAccountName: myapp-sa
      volumes:
        - name: secrets-store
          csi:
            driver: secrets-store.csi.k8s.io
            readOnly: true
            volumeAttributes:
              secretProviderClass: "vault-database-creds"
      containers:
        - name: app
          image: quay.io/example/myapp
          volumeMounts:
            - name: secrets-store
              mountPath: "/mnt/secrets"
              readOnly: true
EOF
```

---

# 🔍 **7. Verifica all’interno del Pod**

Accedi al Pod:

```bash
oc rsh deploy/myapp
```

Lista file:

```bash
ls /mnt/secrets
```

Dovresti vedere:

```
db-user
db-pass
```

Contenuto:

```bash
cat /mnt/secrets/db-user
cat /mnt/secrets/db-pass
```

Se Vault è configurato correttamente, leggerai:

```
myuser
mypassword
```

---


Con questa configurazione:

### ✔ Nessun secret viene memorizzato in OpenShift/etcd

### ✔ Tutti i secret sono cifrati e gestiti in Vault

### ✔ Montaggio dinamico nel filesystem del Pod

### ✔ Rotazione automatica dal lato Vault

### ✔ Compatibile con GitOps (i manifest non contengono segreti)

Riferimenti: 
* https://secrets-store-csi-driver.sigs.k8s.io/introduction
* https://www.redhat.com/en/blog/introducing-the-secret-store-csi-driver-in-openshift


---

