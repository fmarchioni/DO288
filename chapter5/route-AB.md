# Guida Demo: Creare una Route A/B su OpenShift via Console Web

Questa guida mostra come creare una semplice demo di **Route A/B** utilizzando la console web di OpenShift, con due deployment che servono contenuti diversi e un route che distribuisce il traffico tra di essi.

---

## 1. Creare i Deployment

### Deployment 1 (nginx)

1. Vai su **Workloads → Deployments → Create Deployment**
2. Imposta:

   * Name: `nginx-demo`
   * Container Image: `quay.io/nginx/nginx-unprivileged`
3. Clicca **Create**

### Deployment 2 (httpd)

1. Vai su **Workloads → Deployments → Create Deployment**
2. Imposta:

   * Name: `httpd-demo`
   * Container Image: `image-registry.openshift-image-registry.svc:5000/openshift/httpd:latest`
3. Clicca **Create**

---

## 2. Creare i Services

Per ciascun deployment:

1. Vai su **Networking → Services**
2. Clicca **Create Service**
3. Associa il Service al rispettivo Deployment
4. Salva il Service

**Nota:** Avrai due service distinti, ad esempio `nginx-demo` e `httpd-demo`.

---

## 3. Creare la Route A/B

1. Vai su **Networking → Routes**
2. Clicca **Create Route**
3. Imposta:

   * Name: `httpd`
   * Service: seleziona uno dei due service (es. `httpd-demo`)
4. Clicca **Add Alternate Backend**
5. Seleziona l'altro service (es. `nginx-demo`)
6. Imposta la percentuale del traffico per ciascun backend

   * Esempio: `httpd-demo` → 70%, `nginx-demo` → 30%
7. Clicca **Create**

---

## 4. Verifica

1. Copia l'URL della Route creata
2. Apri il browser e ricarica più volte per vedere il traffico distribuito tra i due deployment secondo la percentuale impostata

---

**Risultato:**

* `httpd-demo` e `nginx-demo` servono contenuti diversi
* La Route A/B distribuisce il traffico tra i due servizi secondo le percentuali impostate, permettendo test e confronto diretto.
