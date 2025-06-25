# Guida Tascabile Helm Template

Questa guida fornisce una panoramica delle parole chiave e delle funzioni più utili nei template Helm, basati su Go Template.

---

## ✅ Parole Chiave (Controllo di flusso)

### `if`
Usato per eseguire un blocco di codice solo se una condizione è vera.
```yaml
{{ if .Values.debug }}
env:
  - name: DEBUG
    value: "true"
{{ end }}
```

### `else` / `else if`
Usato per specificare blocchi alternativi in caso la condizione iniziale sia falsa.
```yaml
{{ if .Values.env }}
  value: {{ .Values.env }}
{{ else if .Values.defaultEnv }}
  value: {{ .Values.defaultEnv }}
{{ else }}
  value: "prod"
{{ end }}
```

### `with`
Cambia il contesto corrente (il `.`) a un valore specifico, se esiste.
```yaml
{{ with .Values.image }}
image: {{ .repository | quote }}
imagePullPolicy: {{ .pullPolicy | default "Always" | quote }}
{{ end }}
```

### `range`
Itera su una lista o una mappa. Molto utile per array nei valori.
```yaml
ports:
{{- range .Values.ports }}
  - containerPort: {{ . }}
{{- end }}
```

### `define` e `template`
`define` crea un blocco riutilizzabile. `template` lo include in un punto specifico.
```yaml
{{ define "mychart.labels" }}
app: myapp
version: {{ .Chart.Version }}
{{ end }}

metadata:
  labels:
    {{ include "mychart.labels" . | indent 4 }}
```

### `include`
Include un template definito con `define` e restituisce una stringa.
```yaml
{{ include "mychart.labels" . }}
```

### `block`
Simile a `define`, ma può essere sovrascritto in un chart figlio.
```yaml
{{ block "mychart.content" . }}
Default content
{{ end }}
```

---

## 🛠 Funzioni Utili

Funzioni comuni usate nei template per manipolare valori o formattare output.

| Funzione         | Descrizione                                     | Esempio                                       |
|------------------|--------------------------------------------------|-----------------------------------------------|
| `default`         | Valore di default                               | `{{ .Values.mode | default "prod" }}`         |
| `quote`           | Aggiunge virgolette doppie                      | `{{ .Values.image | quote }}`                 |
| `toYaml`          | Converte un oggetto a YAML                      | `{{ .Values.config | toYaml | indent 4 }}`    |
| `indent`          | Indenta un blocco                               | `{{ include "mytpl" . | indent 2 }}`          |
| `upper` / `lower` | Maiuscole/minuscole                             | `{{ .Values.env | upper }}`                   |
| `replace`         | Sostituisce una stringa                         | `{{ replace "v1" "v2" .Values.version }}`      |
| `printf`          | Format string                                   | `{{ printf "v%s" .Chart.AppVersion }}`        |
| `required`        | Errore se mancante                              | `{{ required "Image obbligatoria" .image }}`  |

---

## 📦 Esempio Completo
Un esempio che usa `define`, `range` e `include` per creare dinamicamente un container.
```yaml
{{- define "mychart.container" -}}
- name: {{ .name }}
  image: {{ .image | quote }}
  ports:
  {{- range .ports }}
  - containerPort: {{ . }}
  {{- end }}
{{- end }}

containers:
{{ include "mychart.container" (dict "name" "app" "image" .Values.image "ports" .Values.ports) | indent 2 }}
```
