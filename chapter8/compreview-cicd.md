# Lab: Implementing Continuous Integration and Deployment with OpenShift Pipelines

## 1. Login su OpenShift e cambio di progetto e directory

```bash
oc login -u developer -p developer https://api.ocp4.example.com:6443
oc project compreview-cicd
cd ~/DO288/labs/compreview-cicd
```

---

## 2. Creazione di un Custom Task

Applichiamo il task `npm`:

```bash
oc apply -f npm-task.yaml
```

**Contenuto di `npm-task.yaml`:**

```yaml
---
apiVersion: tekton.dev/v1
kind: Task
metadata:
  name: npm
spec:
  workspaces:
    - name: source
  params:
    - name: CONTEXT
      type: string
      default: "."
      description: The path where package.json of the project is defined.
    - name: ARGS
      type: string
    - name: NODE_IMAGE
      type: string
      default: "registry.ocp4.example.com:8443/ubi9/nodejs-20-minimal:latest"
  steps:
    - name: npm-run
      image: $(params.NODE_IMAGE)
      script: |
        npm $(params.ARGS)
      workingDir: $(workspaces.source.path)/$(params.CONTEXT)
      env:
        - name: CI
          value: "true"
      securityContext:
        runAsNonRoot: true
        runAsUser: 65532
```

---

## 3. Configurazione della Pipeline

Iniziamo con il file `pipeline.yaml` **incompleto**, che contiene già i parametri ma non i task definiti:

```yaml
---
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: words-cicd-pipeline
spec:
  workspaces:
    - name: shared
  params:
    - name: IMAGE_NAME
      type: string
      default: "words"
    - name: IMAGE_REGISTRY
      type: string
      default: "image-registry.openshift-image-registry.svc:5000"
    - name: GIT_REPO
      type: string
      default: "https://git.ocp4.example.com/developer/DO288-apps"
    - name: GIT_REVISION
      type: string
      default: "main"
    - name: APP_PATH
      type: string
      default: "apps/compreview-cicd/words"
  tasks: []
```

Ora andremo ad aggiungere i task **uno alla volta**.

---

### 3.1 Task: fetch-repository

Aggiungiamo il task per clonare il repository Git:

```yaml
    - name: fetch-repository
      taskRef:
        resolver: cluster
        params:
          - name: kind
            value: task
          - name: name
            value: git-clone
          - name: namespace
            value: openshift-pipelines
      params:
        - name: URL
          value: $(params.GIT_REPO)
        - name: REVISION
          value: $(params.GIT_REVISION)
        - name: DELETE_EXISTING
          value: "true"
        - name: SSL_VERIFY
          value: "false"
      workspaces:
        - name: output
          workspace: shared
```

---

### 3.2 Task: npm-install

Aggiungiamo il task per installare le dipendenze:

```yaml
    - name: npm-install
      taskRef:
        name: npm
        kind: Task
      workspaces:
        - name: source
          workspace: shared
      params:
        - name: CONTEXT
          value: $(params.APP_PATH)
        - name: ARGS
          value: install --no-package-lock
      runAfter:
        - fetch-repository
```

---

### 3.3 Task: npm-test

Aggiungiamo il task per lanciare i test:

```yaml
    - name: npm-test
      taskRef:
        name: npm
        kind: Task
      workspaces:
        - name: source
          workspace: shared
      params:
        - name: CONTEXT
          value: $(params.APP_PATH)
        - name: ARGS
          value: test
      runAfter:
        - npm-install
```

---

### 3.4 Task: npm-lint

Aggiungiamo il task per eseguire la lint analysis:

```yaml
    - name: npm-lint
      taskRef:
        name: npm
        kind: Task
      workspaces:
        - name: source
          workspace: shared
      params:
        - name: CONTEXT
          value: $(params.APP_PATH)
        - name: ARGS
          value: run lint
      runAfter:
        - npm-install
```

---

### 3.5 Task: build-push-image

Aggiungiamo il task per buildare e pushare l’immagine con **Buildah**:

```yaml
    - name: build-push-image
      taskRef:
        resolver: cluster
        params:
          - name: kind
            value: task
          - name: name
            value: buildah
          - name: namespace
            value: openshift-pipelines
      params:
        - name: IMAGE
          value: $(params.IMAGE_REGISTRY)/$(context.pipelineRun.namespace)/$(params.IMAGE_NAME):$(context.pipelineRun.uid)
        - name: DOCKERFILE
          value: ./Containerfile
        - name: CONTEXT
          value: $(params.APP_PATH)
      workspaces:
        - name: source
          workspace: shared
      runAfter:
        - npm-test
        - npm-lint
```

---

### 3.6 Task: oc-deploy

Infine, aggiungiamo il task che esegue il deploy con l’`openshift-client`:

```yaml
    - name: oc-deploy
      taskRef:
        resolver: cluster
        params:
          - name: kind
            value: task
          - name: name
            value: openshift-client
          - name: namespace
            value: openshift-pipelines
      workspaces:
        - name: manifest_dir
          workspace: shared
      params:
        - name: SCRIPT
          value: |
            oc process -f $(params.APP_PATH)/kubefiles/app.yaml \
            -p IMAGE_NAME=$(params.IMAGE_REGISTRY)/$(context.pipelineRun.namespace)/$(params.IMAGE_NAME):$(context.pipelineRun.uid) \
            | oc apply -f -
      runAfter:
        - build-push-image
```

---

### 3.7 Creazione della pipeline

Applichiamo la pipeline completa:

```bash
oc apply -f pipeline.yaml
```

---

## 4. Creazione e collegamento del Secret

```bash
oc apply -f basic-user-pass.yaml
oc secret link pipeline basic-user-pass
```

---

## 5. Avvio della Pipeline

```bash
tkn pipeline start --use-param-defaults words-cicd-pipeline \
  -p APP_PATH=apps/compreview-cicd/words \
  -w name=shared,volumeClaimTemplateFile=volume-template.yaml
```

 
