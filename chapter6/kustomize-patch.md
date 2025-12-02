# Esempio Kustomize OpenShift - Deployment con ConfigMap e patch

Questo esempio mostra come usare **Kustomize** per modificare un Deployment OpenShift, aggiungere un ConfigMap e iniettare i valori nel container.

---

## File: `base/deployment.yml`

**Descrizione:**  
Deployment di base con un container `nginx-unprivileged` esposto sulla porta 8080 e replica iniziale 1.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      run: my-nginx
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - image: quay.io/nginx/nginx-unprivileged
        name: my-nginx
        ports:
        - containerPort: 8080
```

---

## File: `base/kustomization.yml`

**Descrizione:**  
Layer di Kustomize che applica patch al Deployment, genera una ConfigMap e inietta i valori nel container.

```yaml
resources:
- deployment.yml

patches:
  # Patch per aumentare il numero di repliche
  - target:
      group: apps
      version: v1
      kind: Deployment
      name: my-nginx
    path: patch-replicas.yml

  # Patch per iniettare i valori della ConfigMap come variabili d'ambiente
  - target:
      group: apps
      version: v1
      kind: Deployment
      name: my-nginx
    path: patch-env.yml
    
configMapGenerator:
  - name: my-config
    literals:
      - foo=bar
      - hello=world
```

---

## File: `base/patch-env.yml`

**Descrizione:**  
Patch per iniettare tutti i valori della ConfigMap `my-config` come variabili d'ambiente del container.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  template:
    spec:
      containers:
        - name: my-nginx
          envFrom:
            - configMapRef:
                name: my-config
```

---

## File: `base/patch-replicas.yml`

**Descrizione:**  
Patch in formato JSON6902 per modificare il numero di repliche da 1 a 3.

```yaml
- op: replace
  path: /spec/replicas
  value: 3
```

---

## Test del layer base

Per verificare che il layer di base funzioni correttamente senza applicarlo al cluster:

```bash
oc kustomize base
```

Questo comando mostrerà il Deployment finale combinando il file originale, le patch e la ConfigMap generata.

