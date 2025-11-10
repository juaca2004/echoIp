
````markdown
# 🧩 Registro de Correcciones del Chart Helm - EchoIP

Este documento describe los **problemas detectados** y las **correcciones aplicadas** en el chart Helm de **EchoIP**, con el objetivo de garantizar su correcta renderización y despliegue en Kubernetes.

---

## 📂 Archivos Revisados

Se revisaron los siguientes archivos del chart:

- `templates/deployment.yaml`
- `templates/ingress.yaml`
- `templates/service.yaml`

Los demás archivos (`Chart.yaml`, `values.yaml`, `_helpers.tpl`) fueron validados y no presentaron errores de sintaxis.

---

## 1️⃣ deployment.yaml

### 🔍 Problema
Las directivas Helm:
```yaml
{{- with .Values.deployment.annotations }}
{{- end }}
````

estaban **mal ubicadas**.
Interrumpían el bloque `metadata:` antes de cerrarse correctamente, lo que causaba errores de análisis YAML al procesar la plantilla.

### 🛠️ Solución

Se movió el bloque `with` **dentro de `metadata:`** con la indentación adecuada para mantener la estructura jerárquica del documento.

**Antes:**

```yaml
metadata:
  name: {{ include "echoip.fullname" . }}
  labels:
    app.kubernetes.io/name: {{ include "echoip.name" . }}
    ...
{{- with .Values.deployment.annotations }}
  annotations:
{{ toYaml . | indent 4 }}
{{- end }}
```

**Después (corregido):**

```yaml
metadata:
  name: {{ include "echoip.fullname" . }}
  labels:
    app.kubernetes.io/name: {{ include "echoip.name" . }}
    ...
  {{- with .Values.deployment.annotations }}
  annotations:
{{ toYaml . | indent 4 }}
  {{- end }}
```

Esto permite que las anotaciones se apliquen correctamente dentro de la sección `metadata:` sin romper la estructura YAML.

---

## ingress.yaml

### Problema

El archivo comenzaba con la directiva:

```yaml
{{- if .Values.ingress.enabled -}}
```

pero el contenido posterior tenía **indentación irregular** y **listas mal alineadas (`-`)**, lo que provocaba errores de parseo.

Helm requiere que las listas (`-`) estén correctamente indentadas dentro de sus respectivos bloques (`rules`, `tls`, etc.), y que no existan guiones solitarios fuera del contexto YAML.

### 🛠️ Solución

* Se **reordenó la indentación** del archivo.
* Se **alinearon los guiones (`-`)** dentro de las claves `rules:` y `tls:`.
* Se removieron guiones sobrantes y se normalizó el espaciado.

**Ejemplo del bloque corregido:**

```yaml
spec:
  {{- if .Values.ingress.tls }}
  tls:
    {{- range .Values.ingress.tls }}
    - hosts:
        {{- range .hosts }}
        - {{ . | quote }}
        {{- end }}
      secretName: {{ .secretName }}
    {{- end }}
  {{- end }}
  rules:
    {{- range .Values.ingress.hosts }}
    - host: {{ . | quote }}
      http:
        paths:
          - path: {{ $ingressPath }}
            pathType: ImplementationSpecific
            backend:
              service:
                name: {{ $fullName }}
                port:
                  number: {{ $svcPort }}
    {{- end }}
```

Ahora el archivo cumple con la sintaxis YAML esperada y se renderiza correctamente con `helm template`.

---

##service.yaml

###Problema

El bloque `annotations:` tenía **mala indentación**, y el bloque condicional para `service.labels` provocaba un hueco irregular que rompía el formato.

**Antes:**

```yaml
metadata:
  name: {{ include "echoip.fullname" . }}
  labels:
    app.kubernetes.io/name: {{ include "echoip.name" . }}
{{- if .Values.service.labels }}
{{ toYaml .Values.service.labels | indent 4 }}
{{- end }}
  annotations:
{{ toYaml .Values.service.annotations | indent 4 }}
```

Esto hacía que `annotations:` no estuviera correctamente bajo `metadata:`.

### Solución

* Se ajustó la indentación de `labels:` y `annotations:`.
* Se usaron bloques `with` para evitar claves vacías y mantener consistencia con Helm.

**Después (corregido):**

```yaml
metadata:
  name: {{ include "echoip.fullname" . }}
  labels:
    app.kubernetes.io/name: {{ include "echoip.name" . }}
    ...
  {{- with .Values.service.labels }}
{{ toYaml . | indent 4 }}
  {{- end }}
  {{- with .Values.service.annotations }}
  annotations:
{{ toYaml . | indent 4 }}
  {{- end }}
```

Esto garantiza que tanto las etiquetas como las anotaciones se incluyan solo cuando estén definidas en `values.yaml`, respetando la estructura del objeto `metadata`.

---

