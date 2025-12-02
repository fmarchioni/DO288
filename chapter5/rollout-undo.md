# Mini guida: Gestire rollout e rollback con annotazioni in OpenShift

Questa guida mostra come aggiungere una change cause a un Deployment in OpenShift per poter effettuare rollback precisi.

## 1. Aggiungere l'annotazione `kubernetes.io/change-cause`

L'annotazione `kubernetes.io/change-cause` permette di memorizzare una descrizione della modifica effettuata. Questo aiuta a sapere quale versione del Deployment ripristinare in caso di rollback.

```bash
oc annotate deployment myapp kubernetes.io/change-cause="Aggiornamento versione nginx 1.21" --overwrite
```

> L'opzione `--overwrite` permette di aggiornare il valore dell'annotazione se esiste già.

## 2. Eseguire un rollout del Deployment

Per applicare la modifica e creare una nuova revisione:

```bash
oc rollout restart deployment/myapp
```

## 3. Visualizzare lo storico delle revisioni

Per vedere tutte le revisioni del Deployment e le relative change cause:

```bash
oc rollout history deployment/myapp
```

Otterrai un output simile a:

```
REVISION  CHANGE-CAUSE
1         Aggiornamento versione nginx 1.20
2         Aggiornamento versione nginx 1.21
```

## 4. Eseguire un rollback a una revisione specifica

Se vuoi tornare a una versione precedente, puoi usare `--to-revision` specificando il numero della revisione:

```bash
oc rollout undo deployment/myapp --to-revision=1
```

> Questo ripristinerà il Deployment alla revisione 1, corrispondente alla change cause "Aggiornamento versione nginx 1.20".

## 5. Verifica

Dopo il rollback, puoi controllare lo stato del Deployment:

```bash
oc get deployment myapp
oc rollout status deployment/myapp
```

---

### Nota

L'utilizzo delle change cause semplifica la gestione dei rollback e rende più chiaro il motivo di ogni versione del Deployment, migliorando la tracciabilità delle modifiche.
