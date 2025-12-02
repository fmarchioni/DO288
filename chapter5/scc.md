# Guida Demo: Applicare l'SCC anyuid a un Deployment su OpenShift

Questa guida veloce mostra come creare un deployment che parte come root e applicare l'SCC `anyuid` usando la console e la CLI di OpenShift.

---

## 1. Creare il Deployment

Creiamo un deployment chiamato `nginx` con l'immagine ufficiale `nginx` (richiede l'esecuzione come root):

```bash
oc create deployment nginx --image=nginx
```

---

## 2. Creare un ServiceAccount

Creiamo un ServiceAccount dedicato:

```bash
oc create sa my-sa
```

---

## 3. Assegnare l'SCC `anyuid` al ServiceAccount

```bash
oc adm policy add-scc-to-user anyuid -z my-sa
```

* `-z my-sa` indica che stiamo assegnando l'SCC al ServiceAccount `my-sa`.
* Ora `my-sa` può eseguire pod con UID arbitrari, incluso root.

---

## 4. Applicare il ServiceAccount al Deployment

Aggiorniamo il Deployment `nginx` affinché usi il ServiceAccount con `anyuid`:

```bash
oc set serviceaccount deployment/nginx my-sa
```

---

## 5. Verifica

Controlla che i Pod del deployment usino il ServiceAccount corretto:

```bash
oc get deployment nginx -o jsonpath='{.spec.template.spec.serviceAccountName}'
```

Dovrebbe restituire:

```
my-sa
```

---

**Risultato:**
Il deployment `nginx` ora gira come root, grazie all'SCC `anyuid` applicato tramite il ServiceAccount `my-sa`.
