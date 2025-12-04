# Step 1 Creazione Chart Helm per backend e database

## 1) Preparazione ambiente

**1.1 Avvio del Lab e posizionamento nella directory del lab**

```bash
lab start compreview-todo
cd ~/DO288/labs/compreview-todo
```

📝 qui il lab ha già copiato i sorgenti e il Containerfile del frontend.

**1.2 Login al cluster e selezione progetto**

```bash
oc login -u developer -p developer https://api.ocp4.example.com:6443
oc project compreview-todo
```

📝 usare l’utente `developer` (come richiesto dal lab) e lavorare nel progetto `compreview-todo`.

---

## 2) Creare il chart Helm `todo-list` e aggiungere la dipendenza MariaDB

**2.1 Creare il chart base**

```bash
helm create todo-list
cd todo-list
```

📝 `helm create` genera la struttura standard (`Chart.yaml`, `values.yaml`, `templates/`).

**2.2 Dichiarare la dipendenza MariaDB**
Apri `Chart.yaml` e **aggiungi** in fondo (o fai append) la sezione `dependencies:`:

```yaml
dependencies:
  - name: mariadb
    version: 11.3.3
    repository: http://helm.ocp4.example.com/charts
```

Oppure esegui:

```bash
cat <<EOF >> Chart.yaml
dependencies:
  - name: mariadb
    version: 11.3.3
    repository: http://helm.ocp4.example.com/charts
EOF
```

**2.3 Scaricare la dipendenza**

```bash
helm dependency update
```

📝 scarica la chart `mariadb` nel `charts/` locale.

---

## 3) Configurare `values.yaml` del chart todo-list (image, pullPolicy, porta, mariadb, env)

**3.1 Modifica campo image**

Apri `values.yaml` e modifica i valori relativi al campo **image** come segue:

```yaml
image:
  repository: registry.ocp4.example.com:8443/redhattraining/todo-backend
  # This sets the pull policy for images.
  pullPolicy: Always
  # Overrides the image tag whose default is the chart appVersion.
  tag: "release-46"
```
  
In alternativa, le modifiche possono essere fatte anche mediante il tool `sed`:

```bash
# repository -> registry classroom
sed -i "s|repository: .*|repository: registry.ocp4.example.com:8443/redhattraining/todo-backend|g" values.yaml

# forzare Always
sed -i "s|pullPolicy: IfNotPresent|pullPolicy: Always|g" values.yaml

# tag -> release-46
sed -i "s|tag: .*|tag: \"release-46\"|g" values.yaml
```


**3.2 Cambiare la porta dell’app a 3000**
(Se il chart ha `service.port` o `containerPort` impostato, modifica quello giusto; esempio generico:)

```yaml
  port: 3000
```
In alternativa, la modifica può essere fatta anche mediante il tool `sed`:

```bash
sed -i "s/port: 80/port: 3000/g" values.yaml || true
```

**3.3 Aggiungere la configurazione richiesta per la dipendenza `mariadb`**

Append al `values.yaml` il blocco MariaDB (esattamente come richiesto dal lab):

```bash
cat <<EOF>> values.yaml

mariadb:
  auth:
    username: todouser
    password: todopwd
    database: tododb
  primary:
    podSecurityContext:
      enabled: false
    containerSecurityContext:
      enabled: false
  global:
    imageRegistry: "registry.ocp4.example.com:8443"
  image:
    repository: redhattraining/mariadb
    tag: 10.5.10-debian-10-r0

env:
  - name: DATABASE_NAME
    value: tododb
  - name: DATABASE_USER
    value: todouser
  - name: DATABASE_PASSWORD
    value: todopwd
  - name: DATABASE_SVC
    value: todo-list-mariadb
EOF
```

---

## 4) Render dei valori in `templates/deployment.yaml` (iniettare le env nel container)

Per far sì che le env siano applicate al container, modifica il file `templates/deployment.yaml` del chart.

**4.1 Aggiungi le variabili d'ambiente disponibili nel values.yml**

Il file `templates/deployment.yaml` deve contenere nel Container i riferimenti alle variabili d'ambiente recuperate tramite il `values.yml`. Questa è la configurazione aggiornata:

```yaml
      containers:
        - name: {{ .Chart.Name }}
          {{- with .Values.securityContext }}
          env:
            {{- range .Values.env }}
          - name: {{ .name }}
            value: {{ .value }}
            {{- end }}
          env:
            {{- range .Values.env }}
          - name: {{ .name }}
            value: {{ .value }}
            {{- end }}
```
📝 questo ciclo Helm itera su `.Values.env` e crea le env richieste (DATABASE\_USER, DATABASE\_PASSWORD, DATABASE\_NAME, DATABASE\_SVC).

In alternativa, la modifica può essere fatta anche mediante il tool `sed`:

```bash
sed -i '43i\          env:\n            {{- range .Values.env }}\n          - name: {{ .name }}\n            value: {{ .value }}\n            {{- end }}' templates/deployment.yaml
```



---

## 5) Installare il chart su OpenShift

**5.1 Login e cambio di progetto**

Eseguire la login come developer e posizionarsi sul progetto **compreview-todo**:

```bash
oc login -u developer -p developer https://api.ocp4.example.com:6443

oc project compreview-todo
```

📝 installa il chart ed esponi la Route:  

```bash
oc login -u developer -p developer https://api.ocp4.example.com:6443

oc project compreview-todo

helm install todo-list .

oc expose svc/todo-list
```

---

## 6) Verifiche su OpenShift (risorse create e funzionamento)

Dopo l’installazione, verifica le risorse create:

**6.1 Verificare Build / Image / Deployment (se la chart ha creato Deployments)**

```bash
oc get all -n compreview-todo
```

**6.2 Verificare la Build/BuildConfig (se usi `oc new-build`, ma qui helm normalmente crea Deployment direttamente)**

```bash
oc get bc -n compreview-todo
oc get builds -n compreview-todo
```

**6.3 Verificare ImageStream e immagini**

```bash
oc get is -n compreview-todo
```

**6.4 Verificare il servizio MariaDB**

```bash
oc get svc -n compreview-todo | grep mariadb
```

Dovresti vedere un servizio il cui nome risulta `todo-list-mariadb` (il chart, con release `todo-list`, spesso genera servizi con prefisso `todo-list-...`).

**6.5 Verificare la Route esposta per l’API**

```bash
oc get route -n compreview-todo
```

Apri l’URL mostrato per chiamare l’API.

---

## 7) Creare, correggere, buildare e pubblicare la SPA (frontend)

Ora la parte SPA (todo-frontend).

**7.1 Posizionarsi nella cartella del frontend**

```bash
cd ~/DO288/labs/compreview-todo/todo-frontend
```

**7.2 Correggere il `Containerfile` per usare l’utente nginx e la porta 8080**
Apri `Containerfile` e fai due modifiche essenziali:

* assicurati che l’utente sia `nginx` (o uid appropriato)
* fai in modo che nginx ascolti sulla porta 8080 (modifica il default.conf)

Esempio modifiche (aggiungi / sostituisci le parti appropriate nel Containerfile):

```dockerfile
FROM nginx:1.21-alpine

# copia i contenuti statici
COPY dist/ /usr/share/nginx/html

# modifica la config di nginx per ascoltare 8080
RUN sed -i 's/listen       80;/listen       8080;/' /etc/nginx/conf.d/default.conf

# esporre 8080
EXPOSE 8080

# eseguire come utente nginx
USER nginx
```

📝 la `sed` sostituisce `listen 80` con `listen 8080` nella config predefinita; `USER nginx` imposta l’utente.

**7.3 Build e push dell’immagine verso il registry di classroom**
Accertati di avere `podman` o `buildah`/`docker` disponibili. Qui uso `podman` come esempio (in ambienti Red Hat è spesso presente):

```bash
# login al registry (developer:developer)
podman login registry.ocp4.example.com:8443 -u developer -p developer

# build
podman build -t registry.ocp4.example.com:8443/developer/todo-frontend:latest -f Containerfile .

# push (aggiungi --tls-verify=false se il registry di lab richiede di disabilitare TLS verification)
podman push registry.ocp4.example.com:8443/developer/todo-frontend:latest
```

Nota: se non hai podman, puoi usare `buildah bud -f Containerfile -t ...` e poi `buildah push ...` o `docker build`/`docker push` se disponibile.

**7.4 Deploy del frontend su OpenShift e creare la Route**

```bash
# nel progetto compreview-todo
oc new-app registry.ocp4.example.com:8443/developer/todo-frontend:latest --name=todo-frontend
oc expose svc todo-frontend
```

**7.5 Verifica la route del frontend**

```bash
oc get route todo-frontend -o wide
# oppure:
oc get route -n compreview-todo
```

Apri l’URL della route nel browser per verificare l’UI.

**7.6 Test dell’app**
Apri il route del frontend e prova a creare / leggere todo items — l’interfaccia dovrebbe comunicare con l’API (che hai reso esposta in passaggi precedenti).

---

## 8) Controlli finali e troubleshooting rapido

* Se la UI non comunica con l’API: controlla CORS / URL configurato nel frontend (il Containerfile/a config potrebbe contenere l’endpoint dell’API).
* Se l’immagine del backend non viene scaricata: assicurati che il cluster possa accedere al registry `registry.ocp4.example.com:8443` oppure crea `imagePullSecrets` o linka il secret al `ServiceAccount` (`oc secret link pipeline ...` non è necessario qui, ma potresti dover collegare i segreti al SA che esegue i pods).
* Verifica i log del pod API e DB:

```bash
oc logs deployment/<nome-deployment> -n compreview-todo
oc logs <nome-pod-mariadb> -n compreview-todo
```

* Se `helm install` fallisce: guarda `helm status todo-list` e i log `oc get events -n compreview-todo`.

---

# Step 2 Creazione app **todo-frontend**

**2.1 Posizionarsi nella directory del frontend**

```bash
cd ~/DO288/labs/compreview-todo/todo-frontend
```

**2.2 Correggere il `Containerfile`**
Aggiungere l’esposizione della porta 8080 e l’uso dell’utente `nginx`:

```bash
sed -i '29i\\nEXPOSE 8080\n\nUSER nginx' Containerfile
```

**2.3 Effettuare login al registry classroom**

```bash
podman login -u developer -p developer registry.ocp4.example.com:8443
```

**2.4 Build e push dell’immagine**

```bash
podman build . -t registry.ocp4.example.com:8443/developer/todo-frontend:latest
podman push registry.ocp4.example.com:8443/developer/todo-frontend
```

**2.5 Login al cluster e selezione progetto**

```bash
oc login -u developer -p developer https://api.ocp4.example.com:6443
oc project compreview-todo
```

**2.6 Creazione dell’applicazione frontend**

```bash
oc new-app registry.ocp4.example.com:8443/developer/todo-frontend
```

**2.7 Esporre la rotta pubblica**

```bash
oc expose svc/todo-frontend
```

**2.8 Verificare la route**

```bash
oc get route todo-frontend
```

Aprire l’URL nel browser e controllare che l’interfaccia sia disponibile.

---

# Step 3 Deploy app **todo-ssr** usando Source-to-Image (S2I)

**3.1 Login e selezione progetto**

```bash
oc login -u developer -p developer https://api.ocp4.example.com:6443
oc project compreview-todo
```

**3.2 Creare la nuova applicazione S2I da Git**

```bash
oc new-app \
  https://git.ocp4.example.com/developer/DO288-apps \
  --name todo-ssr \
  --context-dir=/apps/compreview-todo/todo-ssr \
  --build-env npm_config_registry="http://nexus-infra.apps.ocp4.example.com/repository/npm"
```

📝 il build S2I scarica le dipendenze NPM dal registry interno Nexus.

**3.3 Creare una ConfigMap con la variabile `API_HOST`**

```bash
oc create configmap todo-ssr-host --from-literal API_HOST="http://todo-list:3000"
```

**3.4 Collegare la ConfigMap al Deployment**

```bash
oc set env deployment/todo-ssr --from cm/todo-ssr-host
```

**3.5 Esporre la rotta pubblica per l’app**

```bash
oc expose svc/todo-ssr
```

**3.6 Verifica della route**

```bash
oc get route todo-ssr
```

Aprire l’URL per controllare che l’app SSR sia attiva e si colleghi correttamente al backend `todo-list`.
