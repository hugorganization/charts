# Guía completa del repo `charts`

Este documento explica **todo** lo que hay en este repositorio: los conceptos de Helm desde cero, qué contiene el `chart-base` variable por variable, cómo viajan esos valores desde un fichero hasta un objeto real de Kubernetes, cómo funciona el pipeline de releases, y el problema que nos encontramos el 17 de julio de 2026 con release-please y cómo se arregló.

Está escrito para poder leerse de arriba abajo sin saber Helm previamente. Si ya lo dominas, ve directo al [índice](#índice).

---

## Índice

1. [La idea en una página](#1-la-idea-en-una-página)
2. [Conceptos básicos de Helm](#2-conceptos-básicos-de-helm)
3. [Estructura del repositorio](#3-estructura-del-repositorio)
4. [El `chart-base` por dentro](#4-el-chart-base-por-dentro)
5. [Dónde viven las variables y cómo viajan](#5-dónde-viven-las-variables-y-cómo-viajan)
6. [Los umbrellas: `atm` y `vnd`](#6-los-umbrellas-atm-y-vnd)
7. [El pipeline de releases](#7-el-pipeline-de-releases)
8. [El incidente del deadlock (17 julio 2026)](#8-el-incidente-del-deadlock-17-julio-2026)
9. [Recetas: comandos del día a día](#9-recetas-comandos-del-día-a-día)
10. [Trampas conocidas](#10-trampas-conocidas)

---

## 1. La idea en una página

Hay **dos repositorios** distintos en juego, y confundirlos es la primera fuente de líos:

| Repositorio | Qué contiene | Rol |
|---|---|---|
| `hugorganization/chart_base` | Un único chart genérico, `chart-base` | La **plantilla reutilizable**. Sabe fabricar un Deployment, un Service, ConfigMaps y un ExternalSecret. No sabe nada de tu aplicación. |
| `hugorganization/charts` | **Este repo.** Los umbrellas `atm` y `vnd` | El **catálogo de microservicios**. No tiene plantillas propias: sólo dice "quiero dos copias de `chart-base`, una configurada así y otra asá". |

La relación es de biblioteca a consumidor. `chart-base` se publica como paquete versionado (`0.1.0`, `0.2.0`, `0.3.0`…) y este repo lo **consume** apuntando a una versión concreta. Cuando `chart-base` saca una versión nueva, aquí no pasa nada hasta que alguien sube el número a mano y ejecuta `helm dependency update`. Eso es deliberado: nadie quiere que un cambio ajeno se cuele en producción sin revisar.

El flujo mental completo:

```
chart_base (otro repo)
    │  publica chart-base-0.3.0.tgz en su Helm repo
    ▼
umbrellas/atm/Chart.yaml   ── "dame chart-base 0.3.0, dos veces"
umbrellas/atm/values.yaml  ── "la copia uno así, la copia dos asá"
    │  helm dependency update  → descarga el .tgz a charts/
    │  helm template / helm install → mezcla plantilla + valores
    ▼
YAML de Kubernetes real (Deployment, Service, ConfigMap…)
```

---

## 2. Conceptos básicos de Helm

Si Helm te suena a chino, esta sección es el mínimo imprescindible. Cada término aparece luego en el resto del documento.

### Chart

Un **chart** es un paquete de Helm: una carpeta con plantillas de YAML y un fichero de valores por defecto. Es el equivalente a un paquete de `npm` o `pip`, pero para Kubernetes.

### Plantilla (template)

Un YAML de Kubernetes con **huecos**. En vez de escribir `image: nginx:1.30.4`, escribes:

```yaml
image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

Todo lo que va entre `{{ }}` se calcula al desplegar. Ese lenguaje se llama Go templates.

### Values

El fichero que **rellena los huecos**. `values.yaml` son los valores por defecto del chart; puedes sobreescribirlos al desplegar. Es el único sitio donde deberías tener que tocar para configurar algo.

### Release

Una **instalación concreta** de un chart en un clúster, con nombre propio. El mismo chart instalado dos veces con nombres distintos son dos releases independientes. El nombre del release aparece en las plantillas como `.Release.Name`, y verás que se usa muchísimo para nombrar objetos.

### Umbrella (chart paraguas)

Un chart que **no tiene plantillas propias**, sólo dependencias. Sirve para agrupar varios servicios y desplegarlos de golpe. `atm` y `vnd` son umbrellas: fíjate en que el `helm lint` avisa `templates/: directory does not exist` — no es un error, es exactamente lo que se espera de un umbrella.

### Subchart y `alias`

Una dependencia de un umbrella es un **subchart**. Lo interesante aquí: puedes incluir **el mismo chart varias veces** dándole un `alias` distinto a cada copia. Eso es justo lo que hacen `atm` y `vnd`:

```yaml
dependencies:
  - name: chart-base
    alias: ms-fake-uno      # copia 1
    version: 0.3.0
  - name: chart-base
    alias: ms-fake-dos      # copia 2
    version: 0.3.0
```

Dos microservicios, una sola plantilla. El `alias` es importante por una razón que se explica en la [sección 5](#5-dónde-viven-las-variables-y-cómo-viajan): **es la clave bajo la que escribes su configuración**.

### `Chart.yaml`, `Chart.lock` y `charts/`

- **`Chart.yaml`** — la ficha del chart: nombre, versión, y qué dependencias quiere. Es lo que tú editas.
- **`Chart.lock`** — la foto de lo que se descargó **de verdad**, con un `digest` (huella criptográfica). Igual que `package-lock.json`. Lo genera Helm, no se toca a mano.
- **`charts/`** — la carpeta donde aterrizan los `.tgz` descargados.

⚠️ En este repo **los `.tgz` no se suben a git**: el `.gitignore` tiene `*.tgz`. Por eso, cuando haces `helm dependency update`, `git status` sólo muestra `Chart.lock` y `Chart.yaml`. **No es un fallo.** El CI reconstruye las dependencias desde cero con `helm dependency build` usando el `Chart.lock` como fuente de verdad.

### `dependency update` vs `dependency build`

Se confunden constantemente:

| Comando | Qué hace | Quién lo usa |
|---|---|---|
| `helm dependency update` | Lee `Chart.yaml`, descarga las versiones que pide, y **reescribe** `Chart.lock` | Tú, en local, cuando cambias una versión |
| `helm dependency build` | Lee `Chart.lock` y descarga **exactamente** eso, ignorando `Chart.yaml` | El CI, para builds reproducibles |

La regla: `update` manda `Chart.yaml → Chart.lock`. `build` obedece al `Chart.lock`.

### Repositorio Helm y GitHub Pages

Un "repo de Helm" es sólo **una URL con un fichero `index.yaml`** que lista qué charts hay y de dónde bajarlos. Aquí se sirve con GitHub Pages desde la rama `gh-pages`. Puedes verlo tú mismo:

```bash
curl -s https://hugorganization.github.io/charts/index.yaml
```

---

## 3. Estructura del repositorio

```
charts/
├── README.md                       # placeholder
├── README2.md                      # este documento
├── .gitignore                      # ignora *.tgz y .idea/
├── release-please-config.json      # cómo se versionan los charts
├── .release-please-manifest.json   # versión actual de cada chart (fuente de verdad)
├── .github/workflows/
│   └── release-please.yml          # el pipeline entero
└── umbrellas/
    ├── atm/
    │   ├── Chart.yaml              # versión de atm + dependencias
    │   ├── Chart.lock              # generado por helm
    │   ├── values.yaml             # ⭐ la configuración de los microservicios
    │   ├── CHANGELOG.md            # generado por release-please
    │   └── charts/                 # .tgz descargados (NO en git)
    └── vnd/
        └── (misma estructura)
```

**El fichero que vas a tocar el 90% de las veces es `umbrellas/<nombre>/values.yaml`.**

Un detalle importante: `.release-please-manifest.json` es **la fuente de verdad de las versiones**. Release-please se fía de este fichero, no del `Chart.yaml`. Si los dos se desincronizan, gana el manifest.

Estado a día de hoy:

```json
{
  "umbrellas/atm": "0.3.0",
  "umbrellas/vnd": "0.1.0"
}
```

---

## 4. El `chart-base` por dentro

Todo lo de esta sección vive en el **otro** repositorio (`hugorganization/chart_base`) y llega aquí empaquetado. Para leerlo en local:

```bash
tar xzf umbrellas/atm/charts/chart-base-0.3.0.tgz -C /tmp
ls /tmp/chart-base/templates/
```

Contenido de la versión **0.3.0**:

```
chart-base/
├── Chart.yaml
├── values.yaml          # ⭐ los valores por defecto
└── templates/
    ├── _helpers.tpl     # funciones de nombrado (no genera nada)
    ├── deployment.yaml
    ├── service.yaml
    ├── serviceaccount.yaml
    ├── configmap-env.yaml
    ├── configmap-file.yaml
    ├── externalsecret.yaml
    └── NOTES.txt        # el mensaje tras instalar
```

### 4.1 `_helpers.tpl` — de dónde salen los nombres

No genera ningún objeto: define **funciones reutilizables**. Es la respuesta a "¿por qué mi Deployment se llama `atm-ms-fake-uno`?".

La función clave es `chart-base.fullname`:

```
{{- define "chart-base.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}
```

Traducido: **`<nombre del release>-<nombre del chart>`**.

Y aquí está el truco que hace que todo esto funcione. Cuando usas un `alias`, **Helm sustituye `.Chart.Name` por el alias**. Así que dentro de la copia aliasada como `ms-fake-uno`, `.Chart.Name` no vale `chart-base`, vale `ms-fake-uno`. Resultado con el release llamado `atm`:

```
atm  +  ms-fake-uno  →  atm-ms-fake-uno
```

Por eso dos copias del mismo chart no se pisan: cada una hereda su alias y genera nombres distintos. Es exactamente lo que se ve al renderizar.

El resto de helpers:

| Helper | Devuelve | Ejemplo real |
|---|---|---|
| `chart-base.name` | El nombre del chart (= el alias) | `ms-fake-uno` |
| `chart-base.fullname` | `<release>-<alias>` | `atm-ms-fake-uno` |
| `chart-base.chart` | `<nombre>-<versión>` para etiquetar | `ms-fake-uno-0.3.0` |
| `chart-base.selectorLabels` | Las etiquetas que casan Deployment ↔ Service | `app.kubernetes.io/name`, `.../instance` |
| `chart-base.labels` | Las anteriores + versión + gestor | |
| `chart-base.serviceAccountName` | El nombre de la ServiceAccount | `atm-ms-fake-uno` |
| `chart-base.envConfigMapName` | `<fullname>-env` | `atm-ms-fake-uno-env` |
| `chart-base.fileConfigMapName` | `<fullname>-file` | `atm-ms-fake-uno-file` |
| `chart-base.secretName` | El nombre del Secret generado | `atm-ms-fake-uno` |

El `trunc 63` que ves en todas partes no es capricho: Kubernetes limita muchos nombres a 63 caracteres por la norma de DNS. Si pones un alias larguísimo, el nombre se corta.

### 4.2 `values.yaml` — el catálogo completo de variables

Estos son **todos** los valores por defecto de `chart-base` 0.3.0, tal cual:

```yaml
replicaCount: 1

image:
  repository: nginx
  pullPolicy: IfNotPresent
  tag: "1.30.4"          # ⚠️ cambió en 0.3.0; antes era "latest"

nameOverride: ""
fullnameOverride: ""

serviceAccount:
  create: true
  automount: true
  annotations: {}
  name: ""

podAnnotations: {}

service:
  type: ClusterIP
  port: 80
  targetPort: 80         # ⚠️ nuevo en 0.3.0
  annotations: {}

containerPort: 80

resources: {}
nodeSelector: {}
tolerations: []
affinity: {}

config:
  env:                   # ConfigMap #1 → variables de entorno
    enabled: true
    data:
      LOG_LEVEL: info
      APP_MODE: production
  file:                  # ConfigMap #2 → fichero montado
    enabled: true
    mountPath: /etc/app
    fileName: app-config.yaml
    content: |
      server:
        port: 8080
      features:
        metrics: true

externalSecret:
  enabled: false
  refreshInterval: 1h
  secretStoreRef:
    name: ""             # obligatorio si enabled=true
    kind: SecretStore    # o ClusterSecretStore
  target:
    name: ""             # vacío ⇒ nombre del release
    creationPolicy: Owner
    deletionPolicy: Retain
    template: {}
  data: []
  dataFrom: []
```

**Los dos cambios de 0.2.0 → 0.3.0 son justo los que motivaron todo el trabajo de esta semana**: se añadió `service.targetPort` y se fijó `image.tag` a `1.30.4`.

### 4.3 `deployment.yaml` — el pod

El objeto principal. Sus piezas:

```yaml
replicas: {{ .Values.replicaCount }}
```
Cuántas copias del pod.

```yaml
image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
```
Ojo a este `| default`: significa "si `image.tag` está **vacío**, usa el `appVersion` del chart". Como en 0.3.0 el tag por defecto es `"1.30.4"` (no está vacío), **el `default` nunca se activa** y `appVersion` no se usa jamás. Esto importa: si algún día quieres que la imagen siga a `appVersion`, tienes que poner `tag: ""` explícitamente.

```yaml
ports:
  - name: http
    containerPort: {{ .Values.containerPort }}
```
El puerto que escucha tu aplicación **dentro** del contenedor. Le pone el nombre `http`.

```yaml
envFrom:
  - configMapRef:
      name: {{ include "chart-base.envConfigMapName" . }}   # si config.env.enabled
  - secretRef:
      name: {{ include "chart-base.secretName" . }}         # si externalSecret.enabled
```
Aquí es donde las variables de entorno y los secretos entran en el contenedor. `envFrom` vuelca **todas** las claves del ConfigMap/Secret como variables de entorno de golpe.

```yaml
annotations:
  checksum/config-env: {{ include (print $.Template.BasePath "/configmap-env.yaml") . | sha256sum }}
```
Un detalle muy elegante que conviene entender: calcula un **hash del contenido del ConfigMap** y lo mete como anotación del pod. ¿Para qué? Porque Kubernetes **no reinicia los pods cuando cambia un ConfigMap**. Al meter el hash en la plantilla del pod, si cambias una variable de entorno el hash cambia, la definición del pod cambia, y Kubernetes hace rolling restart solo. Sin esto, cambiarías `LOG_LEVEL` y no pasaría nada hasta el siguiente reinicio manual.

El resto (`volumeMounts`, `volumes`, `resources`, `nodeSelector`, `tolerations`, `affinity`) se rellena sólo si defines los valores correspondientes.

### 4.4 `service.yaml` — la puerta de entrada

Corto pero es el que nos dio guerra:

```yaml
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.targetPort }}    # ← en 0.2.0 esto era: targetPort: http
      protocol: TCP
      name: http
  selector:
    {{- include "chart-base.selectorLabels" . | nindent 4 }}
```

**La diferencia entre `port` y `targetPort`** es la confusión clásica:

- `port` — el puerto por el que **otros** llaman al Service. La puerta de la calle.
- `targetPort` — el puerto del **contenedor** al que redirige. La puerta interior.
- `containerPort` — el puerto que el contenedor **realmente** abre.

`targetPort` tiene que apuntar a `containerPort`, o el Service reenvía tráfico a un puerto donde no escucha nadie y todo falla con un silencio muy poco informativo.

Ejemplo real de `ms-fake-dos`: `port: 8080` → `targetPort: 80` → el contenedor escucha en el 80. Le llamas al 8080 y por dentro va al 80.

**Este es el cambio de 0.2.0 a 0.3.0.** Antes la plantilla decía `targetPort: http` fijo (el *nombre* del puerto declarado en el Deployment, que Kubernetes resuelve solo). Funcionaba, pero no se podía cambiar. En 0.3.0 pasó a leer `.Values.service.targetPort`, y por eso ahora sí es configurable.

Y el `selector` es lo que une Service ↔ pods: casa por etiquetas, no por nombres.

### 4.5 `serviceaccount.yaml`

La identidad del pod frente al API de Kubernetes. Todo el fichero está envuelto en `{{- if .Values.serviceAccount.create -}}`, así que con `create: false` no se genera nada y el pod usa la ServiceAccount `default` del namespace.

### 4.6 `configmap-env.yaml` — variables de entorno

```yaml
data:
  {{- range $key, $value := .Values.config.env.data }}
  {{ $key }}: {{ $value | quote }}
  {{- end }}
```

Recorre lo que pongas en `config.env.data` y lo convierte en un ConfigMap. El `| quote` fuerza comillas, y eso evita el clásico problema de YAML donde `LOG_LEVEL: no` se interpretaría como el booleano `false`.

Se enchufa al contenedor vía `envFrom`, así que **cada clave se convierte en una variable de entorno con ese nombre exacto**.

### 4.7 `configmap-file.yaml` — fichero de configuración

```yaml
data:
  {{ .Values.config.file.fileName }}: |
    {{- tpl .Values.config.file.content . | nindent 4 }}
```

Mete `config.file.content` en un fichero llamado `config.file.fileName`, que el Deployment monta en `config.file.mountPath` (por defecto `/etc/app/app-config.yaml`), en modo sólo lectura.

Fíjate en `tpl`: significa que **el contenido puede llevar plantillas dentro**. Puedes escribir `{{ .Release.Name }}` dentro de tu fichero de configuración y se sustituirá. Es potente y a la vez fácil de romper si tu config lleva llaves `{{ }}` literales.

### 4.8 `externalsecret.yaml` — secretos desde AWS

El más sofisticado. Requiere tener instalado el operador [External Secrets](https://external-secrets.io/) en el clúster.

La idea: **no guardas secretos en git**. Declaras *dónde* está el secreto (en AWS Secrets Manager, por ejemplo) y el operador lo trae y crea un `Secret` de Kubernetes. El chart-base genera el objeto `ExternalSecret`; el operador hace el trabajo.

```yaml
spec:
  refreshInterval: {{ .Values.externalSecret.refreshInterval | quote }}
  secretStoreRef:
    name: {{ required "externalSecret.secretStoreRef.name es obligatorio cuando externalSecret.enabled=true" .Values.externalSecret.secretStoreRef.name }}
    kind: {{ .Values.externalSecret.secretStoreRef.kind }}
```

Ese `required` es una red de seguridad: si activas `externalSecret.enabled: true` y olvidas el `secretStoreRef.name`, **el render falla con ese mensaje** en vez de generar un objeto roto que fallaría en silencio en el clúster.

Cómo se usa en `atm`/`vnd`:

```yaml
externalSecret:
  enabled: true
  secretStoreRef:
    name: floci-secretsmanager     # el SecretStore ya existente en el clúster
  data:
    - secretKey: DB_PASSWORD       # nombre de la variable que verá el pod
      remoteRef:
        key: prod/app/db           # ruta del secreto en AWS
        property: password         # campo concreto dentro de ese secreto
```

Cadena completa: AWS `prod/app/db` (campo `password`) → el operador crea el Secret `atm-ms-fake-uno` → el Deployment lo carga con `envFrom.secretRef` → el contenedor ve `$DB_PASSWORD`. **`refreshInterval: 1h`** significa que si rotas el secreto en AWS, el Secret se actualiza solo en menos de una hora (aunque el pod no lo relee hasta reiniciar).

`ms-fake-dos` lo tiene a `enabled: false`, así que no genera nada.

### 4.9 `NOTES.txt`

El texto que Helm imprime tras un `install`. No crea nada; sólo te da el `port-forward` listo para copiar.

---

## 5. Dónde viven las variables y cómo viajan

Esta es la sección que más se consulta. **Hay tres sitios donde puede vivir un valor**, y ganan en este orden (de menor a mayor prioridad):

```
1. chart-base/values.yaml        (dentro del .tgz)     ← el default
2. umbrellas/atm/values.yaml     (este repo)           ← tu configuración  ⭐
3. --set / -f en la línea de comandos                  ← lo que gana siempre
```

**El nivel 2 es el tuyo.** El 1 es de otro repo, y el 3 es para emergencias.

### La regla del alias

Aquí está el mecanismo central. En `umbrellas/atm/values.yaml` escribes:

```yaml
ms-fake-uno:            # ← esto NO es un nombre libre: es el ALIAS del Chart.yaml
  service:
    port: 80
```

Todo lo que anides bajo `ms-fake-uno:` se le entrega a esa copia del subchart **como si fuera su `values.yaml` raíz**. Es decir, dentro de la plantilla, `ms-fake-uno.service.port` del umbrella se lee como `.Values.service.port`. La clave de primer nivel desaparece.

```
umbrellas/atm/values.yaml          dentro de la plantilla
─────────────────────────          ─────────────────────
ms-fake-uno:
  service:
    port: 80             ───────►  .Values.service.port
```

⚠️ **Si escribes mal el alias, Helm no te avisa.** Un bloque `ms-fake-tres:` que no exista como alias se ignora en silencio y tus valores no se aplican. Es el error más común y más difícil de ver.

### La sección `global`

```yaml
global:
  ms-fake-uno:
    enabled: true
  ms-fake-dos:
    enabled: true
```

`global` es especial: **se entrega a todos los subcharts a la vez**. Aquí se usa para encender/apagar servicios, conectado con el `condition` del `Chart.yaml`:

```yaml
- name: chart-base
  alias: ms-fake-uno
  condition: global.ms-fake-uno.enabled     # si es false, este subchart no genera nada
```

Poniendo `global.ms-fake-uno.enabled: false` desaparece el microservicio entero del despliegue.

### Mapa completo: de la variable al objeto de Kubernetes

Esta tabla responde a "¿dónde acaba cada cosa?". Todas las rutas son **relativas al bloque del alias** (o sea, `service.port` significa `ms-fake-uno.service.port` en el values del umbrella).

| Variable | Plantilla que la lee | Dónde acaba |
|---|---|---|
| `replicaCount` | `deployment.yaml` | `spec.replicas` |
| `image.repository` | `deployment.yaml` | parte izquierda de `image:` |
| `image.tag` | `deployment.yaml` | parte derecha de `image:` (si vacío ⇒ `appVersion`) |
| `image.pullPolicy` | `deployment.yaml` | `imagePullPolicy` |
| `containerPort` | `deployment.yaml` | `containerPort` del puerto llamado `http` |
| `podAnnotations` | `deployment.yaml` | anotaciones del pod |
| `resources` | `deployment.yaml` | `resources` (límites de CPU/RAM) |
| `nodeSelector`/`tolerations`/`affinity` | `deployment.yaml` | reglas de en qué nodo cae el pod |
| `service.type` | `service.yaml` | `ClusterIP` / `NodePort` / `LoadBalancer` |
| `service.port` | `service.yaml` | puerto expuesto del Service |
| `service.targetPort` | `service.yaml` | puerto destino → **debe casar con `containerPort`** |
| `service.annotations` | `service.yaml` | anotaciones del Service |
| `serviceAccount.create` | `serviceaccount.yaml` | si se crea o no |
| `serviceAccount.name` | `_helpers.tpl` | nombre; vacío ⇒ el `fullname` |
| `serviceAccount.automount` | `serviceaccount.yaml` | `automountServiceAccountToken` |
| `config.env.enabled` | `configmap-env.yaml` | si existe el ConfigMap de entorno |
| `config.env.data` | `configmap-env.yaml` | **cada clave ⇒ una variable de entorno** |
| `config.file.enabled` | `configmap-file.yaml` | si existe el ConfigMap de fichero |
| `config.file.content` | `configmap-file.yaml` | contenido del fichero (admite plantillas) |
| `config.file.fileName` | `configmap-file.yaml` | nombre del fichero |
| `config.file.mountPath` | `deployment.yaml` | carpeta donde se monta |
| `externalSecret.enabled` | `externalsecret.yaml` | si se crea el ExternalSecret |
| `externalSecret.secretStoreRef.name` | `externalsecret.yaml` | **obligatorio** si `enabled: true` |
| `externalSecret.data[]` | `externalsecret.yaml` | qué secretos traer y con qué nombre |
| `externalSecret.refreshInterval` | `externalsecret.yaml` | cada cuánto se resincroniza |
| `nameOverride` / `fullnameOverride` | `_helpers.tpl` | cambian el nombre de **todos** los objetos |

### Ejemplo completo, de punta a punta

Partimos de esto en `umbrellas/atm/values.yaml`:

```yaml
ms-fake-dos:
  service:
    port: 8080
  config:
    env:
      enabled: true
      data:
        APP_MODE: dos
        LOG_LEVEL: info
```

Lo que ocurre:

1. `service.port: 8080` sobreescribe el default `80` de `chart-base`.
2. `service.targetPort` **no se define** → se queda con el default `80` de `chart-base`.
3. `image` **no se define** → hereda `nginx:1.30.4` del default de 0.3.0.
4. El alias `ms-fake-dos` + release `atm` → todos los objetos se llaman `atm-ms-fake-dos`.

Y sale esto:

```yaml
kind: Service
metadata:
  name: atm-ms-fake-dos
spec:
  ports:
    - port: 8080
      targetPort: 80
---
kind: Deployment
metadata:
  name: atm-ms-fake-dos
spec:
  template:
    spec:
      containers:
        - image: "nginx:1.30.4"
          ports:
            - name: http
              containerPort: 80
          envFrom:
            - configMapRef:
                name: atm-ms-fake-dos-env      # contiene APP_MODE=dos, LOG_LEVEL=info
```

Compruébalo tú mismo cuando quieras:

```bash
helm template atm umbrellas/atm
```

---

## 6. Los umbrellas: `atm` y `vnd`

**Ambos son idénticos**: sus `values.yaml` son byte a byte iguales. Cada uno define dos microservicios de mentira sobre `nginx`, pensados para probar el pipeline.

| | `ms-fake-uno` | `ms-fake-dos` |
|---|---|---|
| Imagen | `nginx:1.30.4` (heredada) | `nginx:1.30.4` (heredada) |
| `service.port` | 80 | 8080 |
| `service.targetPort` | 80 (explícito) | 80 (heredado) |
| `APP_MODE` | `uno` | `dos` |
| `LOG_LEVEL` | `debug` | `info` |
| ExternalSecret | ✅ activo (`DB_PASSWORD`) | ❌ desactivado |

Versiones actuales:

| Chart | Versión | chart-base |
|---|---|---|
| `atm` | 0.3.0 ✅ publicado | 0.3.0 |
| `vnd` | 0.2.0 ✅ publicado | 0.3.0 |

### Qué cambiamos y por qué

El cambio de esta semana fue **subir ambos umbrellas de chart-base 0.2.0 a 0.3.0**, y aprovechar las dos novedades de esa versión:

1. **Añadir `service.targetPort: 80`** a `ms-fake-uno`. En 0.2.0 esta variable **no existía** y la plantilla tenía `targetPort: http` fijo. Un intento previo de configurarla en 0.2.0 no habría hecho nada: no era un error visible, simplemente config muerta que ninguna plantilla leía.

2. **Quitar los bloques `image`** de ambos servicios. Antes hacía falta fijar `nginx:1.30.4` a mano porque **el default de 0.2.0 era `tag: "latest"`**. En 0.3.0 el default ya es `1.30.4`, así que el bloque sobra y el resultado renderizado es idéntico.

⚠️ **Este segundo punto era delicado.** Quitar los bloques `image` *estando aún en chart-base 0.2.0* habría despinado las dos imágenes de `1.30.4` a `latest` sin previo aviso: un tag flotante en un values de producción. El cambio sólo es seguro **junto con** la subida a 0.3.0. Las dos piezas van atadas y no deben separarse en commits distintos.

---

## 7. El pipeline de releases

Todo vive en `.github/workflows/release-please.yml` y se dispara **en cada push a `main`**.

### Las dos herramientas y su reparto

| Herramienta | De qué manda |
|---|---|
| **release-please** | De la **versión**: decide el número, edita `Chart.yaml`, escribe el `CHANGELOG.md` y el manifest |
| **chart-releaser** | De la **publicación**: empaqueta el `.tgz`, crea el GitHub release y actualiza el `index.yaml` de `gh-pages` |

Que sean dos herramientas con responsabilidades separadas es la clave para entender el incidente de la sección 8.

### Conventional commits: el prefijo decide la versión

Release-please lee **el prefijo de tus mensajes de commit** para decidir cuánto subir la versión. No es cosmético:

| Prefijo | Efecto | Ejemplo |
|---|---|---|
| `feat:` | sube la **minor** (0.2.0 → 0.3.0) | `feat: bump chart-base to 0.3.0` |
| `fix:` | sube la **patch** (0.3.0 → 0.3.1) | `fix: corregir targetPort` |
| `feat!:` o `BREAKING CHANGE:` | sube la **major** (0.3.0 → 1.0.0) | |
| `chore:`, `docs:`, `ci:` | **no publica nada** | |

Además, release-please **reparte los commits por carpeta**: un commit que sólo toca `umbrellas/vnd/` sube la versión de `vnd` y deja `atm` en paz. Por eso conviene no mezclar los dos umbrellas en un mismo commit.

### El ciclo completo

```
1. Haces push de "feat: ..." a main
        ▼
2. release-please abre (o actualiza) una PR "chore: release main"
   con el bump de versión + CHANGELOG + manifest
        ▼
3. Tú mergeas esa PR    ← nada se publica hasta que hagas esto
        ▼
4. chart-releaser empaqueta, crea el release y actualiza gh-pages
        ▼
5. El chart queda instalable desde https://hugorganization.github.io/charts
```

**Nada se publica solo.** La PR de release es el punto de control humano.

### Los tres jobs

```yaml
jobs:
  tag-release-pr:    # ← añadido para arreglar el deadlock (sección 8)
  release-please:    # needs: tag-release-pr
  publish:           # chart-releaser
```

---

## 8. El incidente del deadlock (17 julio 2026)

Merece su propia sección porque es **un fallo que no daba ningún error** y puede volver a aparecer con otra cara.

### El síntoma

Se hizo push de un `feat:` correcto a `main`. El workflow salió **verde**. Pero la versión de `atm` no subía y no aparecía ninguna PR de release. Todo parecía bien y no funcionaba nada.

### La causa

En el log de release-please, enterradas al final, estas dos líneas:

```
❯ Found pull request #1: 'chore: release main'
⚠ There are untagged, merged release PRs outstanding - aborting
```

La cadena causal completa:

1. `release-please-config.json` tiene **`"skip-github-release": true"`** (porque de publicar se encarga chart-releaser, no release-please). En el log se ve como `✔ Release skipped from strategy config`.
2. Pero ese paso de release que se está saltando es **el único que reetiqueta sus PRs** de `autorelease: pending` a `autorelease: tagged`.
3. Así que la PR #1, mergeada el 16 de julio, se quedó **para siempre** en `autorelease: pending`.
4. En cada ejecución posterior, release-please veía una PR de release mergeada "sin taggear", asumía que un release anterior quedó a medias, y **abortaba antes de crear nada**.
5. Como abortaba, nunca reetiquetaba. Como nunca reetiquetaba, siempre abortaba. **Deadlock permanente.**

Lo que más despistaba: **los tags `atm-0.2.0` sí existían en el repo**. Pero los había creado chart-releaser, no release-please, y release-please no los reconoce como suyos. Todo parecía correcto desde fuera.

### Por qué no bastaba con quitar `skip-github-release`

Fue lo primero que se investigó, y no funciona. Chart-releaser deduce "¿ya publiqué esta versión?" mirando **si existe un GitHub release con ese nombre exacto**:

```go
if r.config.SkipExisting {
    existingRelease, _ := r.github.GetRelease(context.TODO(), releaseName)
    if existingRelease != nil {
        continue
    }
}
```

Release-please taggearía `atm-0.3.0` y chart-releaser nombra su release `atm-0.3.0`: **el mismo nombre**. Si release-please crea el release primero, chart-releaser lo ve, cree que ya está publicado, y ese `continue` se salta el chart entero: no sube el `.tgz` ni actualiza el índice. La versión se tagearía pero **nunca sería instalable**.

También se descartó `packages_with_index` como escapatoria (el `continue` ocurre *antes* de copiar el paquete a la rama) y `skip_upload`, que está engañosamente nombrado: en `cr.sh` **no** salta `cr upload`, salta `cr index`.

### La solución aplicada (opción A)

Se mantuvo el reparto (release-please versiona, chart-releaser publica) y se añadió un job que hace **el reetiquetado que release-please no hace**:

```yaml
tag-release-pr:
  runs-on: ubuntu-22.04
  permissions:
    pull-requests: write
  steps:
    - name: Marcar PRs de release mergeadas como tagged
      env:
        GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        GH_REPO: ${{ github.repository }}
      run: |
        gh label create "autorelease: tagged" --color ededed \
          --description "Release PR has been tagged" 2>/dev/null || true
        for pr in $(gh pr list --state merged --label "autorelease: pending" \
                      --json number --jq '.[].number'); do
          echo "Reetiquetando PR #$pr"
          gh pr edit "$pr" --add-label "autorelease: tagged" \
                           --remove-label "autorelease: pending"
        done

release-please:
  needs: tag-release-pr      # ← el desatasco ocurre siempre ANTES
```

Dos decisiones importantes:

- **`--state merged`**: sólo toca PRs ya mergeadas. La PR de release **abierta** debe conservar su `pending`, o release-please perdería la pista de su propia PR.
- **`needs: tag-release-pr`**: garantiza que la limpieza ocurre antes de que release-please evalúe el estado, en la misma ejecución.

### Verificado

- El log muestra `Reetiquetando PR #2` y la PR quedó en `autorelease: tagged`.
- Cero apariciones de `aborting`.
- `atm-0.3.0` publicado y sirviéndose en el `index.yaml` de gh-pages.
- El siguiente `feat:` (el de `vnd`) generó la PR #3 **solo, sin tocar nada a mano**.

### Alternativas descartadas

- **B — `include-v-in-tag: true`**: release-please taggearía `atm-v0.3.0`, sin colisión con el `atm-0.3.0` de chart-releaser. Funciona, pero deja **dos GitHub releases por versión** para siempre.
- **C — jubilar chart-releaser** y publicar con `helm package` + `helm repo index` a mano. Es lo más limpio de verdad (una sola herramienta manda) y sigue sobre la mesa si el repo crece, pero implica migrar el índice existente con cuidado.

---

## 9. Recetas: comandos del día a día

### Subir la versión de chart-base en un umbrella

```bash
# 1. Edita la versión en las dependencias
vim umbrellas/atm/Chart.yaml        # version: 0.3.0 → 0.4.0

# 2. Descarga y regenera el lock
helm dependency update umbrellas/atm

# 3. Comprueba QUÉ cambia de verdad antes de fiarte
helm template atm umbrellas/atm | grep -E "image:|targetPort:"
helm lint umbrellas/atm

# 4. Commit con feat: para que release-please haga su trabajo
git add umbrellas/atm/Chart.yaml umbrellas/atm/Chart.lock
git commit -m "feat: bump chart-base to 0.4.0 en atm"
git push origin main

# 5. Mergea la PR "chore: release main" que aparecerá
```

### Ver el YAML final sin desplegar nada

```bash
helm template atm umbrellas/atm                    # todo
helm template atm umbrellas/atm | grep -A15 "kind: Service"
```

### Leer el chart-base por dentro

```bash
tar xzf umbrellas/atm/charts/chart-base-0.3.0.tgz -C /tmp
cat /tmp/chart-base/values.yaml
cat /tmp/chart-base/templates/service.yaml
```

### Comparar dos versiones de chart-base

Lo más útil cuando subes de versión y quieres saber qué te va a cambiar:

```bash
helm template atm umbrellas/atm > /tmp/nuevo.yaml
git stash && helm dependency update umbrellas/atm
helm template atm umbrellas/atm > /tmp/viejo.yaml
git stash pop && helm dependency update umbrellas/atm
diff /tmp/viejo.yaml /tmp/nuevo.yaml
```

### Comprobar qué hay publicado

```bash
curl -s https://hugorganization.github.io/charts/index.yaml
gh release list
gh pr list --state open
```

### Diagnosticar el pipeline cuando "va verde pero no hace nada"

```bash
gh run list --limit 5
gh run view <ID> --log | grep -iE "aborting|skipped|Considering"
gh pr list --state merged --label "autorelease: pending"    # debe salir vacío
```

Ese último comando es **el chequeo de salud del deadlock**. Si devuelve algo, release-please está atascado.

### Instalar de verdad

```bash
helm repo add mis-charts https://hugorganization.github.io/charts
helm repo update
helm install atm mis-charts/atm -n mi-namespace --create-namespace
```

---

## 10. Trampas conocidas

Recopilación de todo lo que nos mordió, para no repetirlo.

**El alias mal escrito se ignora en silencio.** Un bloque bajo un nombre que no existe como `alias` no da error: simplemente tu configuración no se aplica. Si un valor "no hace efecto", revisa el alias antes que nada.

**Configurar una variable que la plantilla no lee tampoco da error.** Nos pasó con `targetPort` en chart-base 0.2.0: la plantilla tenía `targetPort: http` fijo y jamás miraba el values. La línea existía y no hacía nada. Ante la duda: `grep` la variable en las plantillas del `.tgz`, o `helm template` y mira el resultado.

**`targetPort` debe casar con `containerPort`.** Si no, el Service reenvía a un puerto muerto y falla en silencio.

**Quitar `image` de los values te deja a merced del default del chart-base.** En 0.2.0 ese default era `latest`; en 0.3.0 es `1.30.4`. Quitar el bloque estando en 0.2.0 despinaba la imagen a un tag flotante sin avisar.

**El `| default .Chart.AppVersion` del tag casi nunca se activa.** Sólo funciona si `image.tag` está **vacío**. Con el default actual (`"1.30.4"`), `appVersion` es decorativo.

**Los `.tgz` no están en git** (`.gitignore`). Que `git status` no los mencione tras un `dependency update` es lo normal, no un fallo.

**`helm lint` avisa `templates/: directory does not exist` en los umbrellas.** Es esperado: un umbrella no tiene plantillas propias.

**`update` vs `build`.** `dependency update` reescribe el lock desde `Chart.yaml`; `dependency build` obedece al lock. En local usas `update`; el CI usa `build`.

**El prefijo del commit decide la versión.** Un `chore:` no publica nada. Si tu cambio debe salir, usa `feat:` o `fix:`.

**No mezcles umbrellas en un commit.** Release-please reparte por carpeta; un commit que toca `atm` y `vnd` a la vez enturbia los dos changelogs.

**Un workflow en verde no significa que haya funcionado.** El deadlock de la sección 8 salía verde en todas las ejecuciones. Verifica el efecto (¿hay PR?, ¿subió la versión?, ¿está en el `index.yaml`?), no el color.

**`skip_upload` de chart-releaser no salta `cr upload`, salta `cr index`.** Está mal nombrado; no lo uses esperando lo contrario.

---

## Estado a 20 de julio de 2026

| | Estado |
|---|---|
| `atm` | **0.3.0** publicado, sobre chart-base 0.3.0 |
| `vnd` | **0.2.0** publicado, sobre chart-base 0.3.0 |
| Pipeline | Sano — verificado en un ciclo completo sin intervención manual |
| Deuda pendiente | Valorar la opción C (jubilar chart-releaser) si el repo crece |

> Nota: si tu copia local muestra `vnd` en `0.1.0`, es que te falta un `git pull` — el release `vnd-0.2.0` ya está publicado.
