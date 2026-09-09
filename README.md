# k8s-apps

Manifiestos de Kubernetes para el cluster **k3s** de la Raspberry Pi, gestionados
con **Argo CD** (GitOps).

> **Este repo es público.** No metas aquí `.env`, tokens, contraseñas ni nada
> sensible. Lo sensible sigue en `infra/`, que es local y privado.

---

## La idea de GitOps en una frase

El repo es la fuente de la verdad. Tú **no** haces `kubectl apply`: haces commit
y Argo CD lo aplica por ti. Si algo en el cluster se desvía del repo, Argo lo
devuelve a su sitio.

```
   editas YAML  →  git push  →  Argo lo detecta  →  aplica al cluster
```

---

## Estructura

```
k8s-apps/
├── apps/
│   └── podinfo/            ← la app: qué se despliega
│       ├── namespace.yaml
│       ├── deployment.yaml ← pods, imagen, réplicas, probes
│       ├── service.yaml    ← IP interna estable para los pods
│       └── ingress.yaml    ← entrada desde fuera (vía Traefik)
└── argocd/
    └── podinfo-app.yaml    ← la Application: le dice a Argo qué vigilar
```

La distinción importante: lo de `apps/` lo gestiona Argo. Lo de `argocd/` se
aplica **una sola vez a mano**, para dar de alta la app en Argo.

---

## Puesta en marcha

### 1. Ajustar la URL del repo

Edita `argocd/podinfo-app.yaml` y pon tu URL real en `repoURL`.

### 2. Dar de alta la Application (solo esta vez, a mano)

```bash
export KUBECONFIG=$HOME/.kube/config
kubectl apply -f argocd/podinfo-app.yaml
```

### 3. Ver qué hace

```bash
kubectl get application podinfo -n argocd
kubectl get pods -n podinfo
```

Estado esperado: `Synced` / `Healthy`.

---

## Cómo entrar a podinfo

En esta Raspberry, **Pi-hole ocupa los puertos 80 y 443 del host**, así que
Traefik no puede servir ahí. Se entra por su NodePort: **32602**.

Además el Ingress enruta **por nombre de host**, no por IP: hay que pedir
explícitamente `podinfo.local`.

**Comprobación rápida (desde la Pi):**

```bash
curl -H "Host: podinfo.local" http://192.168.1.12:32602
```

**Desde el navegador del portátil**, añade a tu fichero `hosts`:

```
192.168.1.12  podinfo.local
```

y abre `http://podinfo.local:32602`.

**Alternativa sin tocar nada** (port-forward, ocupa la terminal):

```bash
kubectl port-forward -n podinfo svc/podinfo 9898:80 --address 0.0.0.0
# → http://192.168.1.12:9898
```

---

## El ejercicio que enseña GitOps

1. Abre `apps/podinfo/deployment.yaml` y cambia `replicas: 2` por `replicas: 3`
2. `git commit` + `git push`
3. Mira: `kubectl get pods -n podinfo -w`

Aparece un tercer pod sin que tú hayas hecho ningún `kubectl apply`. Eso es todo.

**Ahora al revés**, para ver el `selfHeal`:

```bash
kubectl scale deploy/podinfo -n podinfo --replicas=1
kubectl get pods -n podinfo -w
```

Argo detecta que el cluster no coincide con el repo y lo devuelve a 3. El repo
manda. Por defecto Argo revisa cada ~3 minutos, así que ten paciencia.

---

## Notas del cluster

| | |
|---|---|
| k3s | v1.35.4+k3s1, nodo único `raspberrypi` |
| Argo CD | v3.3.9, namespace `argocd` |
| Ingress | Traefik (incluido en k3s) — NodePort HTTP 32602, HTTPS 30928 |
| Storage | `local-path-provisioner` (incluido en k3s) |

⚠️ `kubectl` en esta máquina es el binario de k3s y busca por defecto un fichero
que es root-only. Sin esto falla con *permission denied*:

```bash
export KUBECONFIG=$HOME/.kube/config
```

---

## Acceso a la UI de Argo CD

`argocd-server` tiene Ingress (`argocd.regata.local`, mismo NodePort 32602
que el resto). Añade `192.168.1.12 argocd.regata.local` (o `100.107.90.9`
por Tailscale) a tu `hosts` y abre:

```
http://argocd.regata.local:32602
```

Usuario `admin`, contraseña:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

⚠️ **Este Ingress vive fuera del flujo GitOps de este repo**: la instalación
de Argo CD en sí no está gestionada por ninguna `Application` (no puede
gestionarse a sí misma con esta configuración tan simple) — se instaló a
mano con el manifest oficial upstream (`infra/kube_3ks/comandos.md`, repo
privado). El Ingress (`argocd/argocd-server-ingress.yaml`) y el flag
`--insecure` que necesita `argocd-server` para servir HTTP plano detrás de
Traefik están versionados aquí solo para no perder el rastro, pero se
aplican **a mano** — ver los comentarios de ese fichero.

**Alternativa sin Ingress** (por si el DNS/`hosts` falla, o prefieres no
exponer el login a toda la LAN): `~/scripts/argocd-ui.sh` en la Pi levanta
un `kubectl port-forward` a demanda.

---

## `metrics-api`

Segundo microservicio del cluster (el primero con lógica propia, no una demo). Es una
API Spring Boot que consulta InfluxDB y resume CPU/temperatura/RAM de la Pi en
`GET /api/metrics/resumen`. Código fuente en
`~/Documents/projects/_developing/microservicios/metrics-api/` (repo separado — este
repo `k8s-apps` es solo manifiestos).

Sin imagen en ningún registry todavía: se construye y se importa a mano en la Pi.

```bash
cd ~/Documents/projects/_developing/microservicios/metrics-api
docker build -t metrics-api:0.1.0 .
docker save metrics-api:0.1.0 | sudo k3s ctr images import -
```

### El único paso que nunca va al repo: el `Secret`

`configmap.yaml` lleva `INFLUX_URL`/`INFLUX_ORG`/`INFLUX_BUCKET` (no sensibles). El
token sí lo es, y este repo es público, así que el `Secret` se crea a mano una vez,
directamente en el cluster (nunca como YAML en git):

```bash
export KUBECONFIG=$HOME/.kube/config
kubectl create secret generic metrics-api-influx \
  --from-literal=INFLUX_TOKEN=<el-token-de-infra/docker/metricas/monitoring/.env> \
  -n metrics-api
```

Si el namespace `metrics-api` aún no existe (primer despliegue), créalo antes o espera
a que Argo lo cree (`CreateNamespace=true`) y repite el `kubectl create secret`.

### Dar de alta la Application (solo la primera vez)

```bash
kubectl apply -f argocd/metrics-api-app.yaml
```

### Probar

```bash
kubectl get application metrics-api -n argocd     # Synced / Healthy
```

**Vía Ingress** (mismo NodePort 32602 que `podinfo`, enrutado por host):

```bash
curl -H "Host: metrics-api.local" http://192.168.1.12:32602/api/metrics/resumen
```

O añadiendo `192.168.1.12 metrics-api.local` al `hosts` del portátil y abriendo
`http://metrics-api.local:32602/api/metrics/resumen` en el navegador.

**Alternativa sin Ingress** (port-forward, ocupa la terminal):

```bash
kubectl port-forward -n metrics-api svc/metrics-api 8090:80 --address 0.0.0.0
curl http://192.168.1.12:8090/api/metrics/resumen
```

---

## `ingest-service` (regata-platform)

Primer microservicio de `regata-platform`: API FastAPI que genera una regata
simulada y escribe su telemetría en la InfluxDB propia del proyecto (bucket
`regatas`, puerto 8088 — distinta de la de `metrics-api`). Código fuente en
`~/Documents/projects/regata-platform/` (repo separado, privado). Manifiestos
aquí bajo `apps/regata/ingest-service/`, namespace **`regatas`** — los demás
servicios del mismo proyecto (`session-service`, `analysis-service`) comparten
namespace y viven junto a este en `apps/regata/<servicio>/`.

Sin imagen en ningún registry todavía. Ojo: a diferencia de `metrics-api`, el
contexto de build es la **raíz** del monorepo `regata-platform`, no la carpeta
del servicio — depende de paquetes locales hermanos (`libs/telemetry-schema`,
`tools/simulator`; ver ADR 0007 de ese repo):

```bash
cd ~/Documents/projects/regata-platform
docker build -f services/ingest-service/Dockerfile -t ingest-service:0.1.0 .
docker save ingest-service:0.1.0 | sudo k3s ctr images import -
```

### El único paso que nunca va al repo: el `Secret`

`configmap.yaml` lleva `INFLUXDB_URL`/`INFLUXDB_ORG`/`INFLUXDB_BUCKET` (no
sensibles). El token sí lo es:

```bash
export KUBECONFIG=$HOME/.kube/config
kubectl create secret generic regata-ingest-service-influx \
  --from-literal=INFLUXDB_TOKEN=<el-token-de-regata-platform/.env> \
  -n regatas
```

Si el namespace `regatas` aún no existe, créalo antes o espera a que Argo lo
cree (`CreateNamespace=true`) y repite el `kubectl create secret`.

### Dar de alta la Application (solo la primera vez)

```bash
kubectl apply -f argocd/regata-ingest-service-app.yaml
```

### Probar

```bash
kubectl get application regata-ingest-service -n argocd     # Synced / Healthy
```

**Vía Ingress** (mismo NodePort 32602, enrutado por host):

```bash
curl -X POST -H "Host: ingest.regata.local" -H "Content-Type: application/json" \
  http://192.168.1.12:32602/ingest/simulated -d '{}'
```

**Alternativa sin Ingress** (port-forward, ocupa la terminal):

```bash
kubectl port-forward -n regatas svc/ingest-service 8091:80 --address 0.0.0.0
curl -X POST http://192.168.1.12:8091/ingest/simulated -H "Content-Type: application/json" -d '{}'
```

---

## `session-service` (regata-platform)

Segundo microservicio de `regata-platform`: API Spring Boot para metadatos de
barcos/sesiones (Postgres). Código fuente en
`~/Documents/projects/regata-platform/` (repo separado, privado). Manifiestos
aquí bajo `apps/regata/session-service/`, mismo namespace `regatas` que
`ingest-service`.

A diferencia de `ingest-service`, aquí **Postgres se despliega dentro del
propio cluster** (`postgres.yaml`: `StatefulSet` + `Service` headless + PVC de
1Gi vía `local-path-provisioner`) en vez de reutilizar algo que ya corría en
Docker en el host — es el primer `StatefulSet`/PVC del cluster (ADR 0004 de
`regata-platform`).

También a diferencia de `ingest-service`, el contexto de build es la **propia
carpeta del servicio**, no la raíz del monorepo: es un proyecto Maven
autocontenido, sin paquetes hermanos de los que depender (ADR 0007 de
`regata-platform` no aplica aquí).

```bash
cd ~/Documents/projects/regata-platform/services/session-service
docker build -t session-service:0.1.0 .
docker save session-service:0.1.0 | sudo k3s ctr images import -
```

### El único paso que nunca va al repo: el `Secret`

`configmap.yaml` lleva `POSTGRES_DB`/`SPRING_DATASOURCE_URL` (no sensibles). El
usuario y la contraseña de Postgres sí lo son, y los usan **dos** manifiestos
(el `StatefulSet` de Postgres y el `Deployment` de `session-service`) a partir
del mismo `Secret`:

```bash
export KUBECONFIG=$HOME/.kube/config
kubectl create secret generic session-service-postgres \
  --from-literal=POSTGRES_USER=regata \
  --from-literal=POSTGRES_PASSWORD=<elige-una-contraseña-nueva-para-produccion> \
  -n regatas
```

Si el namespace `regatas` aún no existe, créalo antes o espera a que Argo lo
cree (`CreateNamespace=true`) y repite el `kubectl create secret`.

⚠️ Usa una contraseña **distinta** de la de `.env`/`docker-compose.yml` de
`regata-platform` (esa es solo para dev local) — esta es la de producción, en
el cluster real.

### Dar de alta la Application (solo la primera vez)

```bash
kubectl apply -f argocd/regata-session-service-app.yaml
```

### Probar

```bash
kubectl get application regata-session-service -n argocd     # Synced / Healthy
kubectl get pods -n regatas                                  # session-service-postgres-0 y session-service
```

⚠️ En el primer despliegue es normal ver `session-service` reiniciar una o dos
veces (`CrashLoopBackOff` breve) mientras `session-service-postgres-0` termina
de arrancar — no hay orden de arranque garantizado entre los dos manifiestos,
y `session-service` necesita Postgres arriba para migrar el esquema con
Flyway. Se estabiliza solo.

**Vía Ingress** (mismo NodePort 32602, enrutado por host):

```bash
curl -H "Host: session.regata.local" http://192.168.1.12:32602/health
curl -X POST -H "Host: session.regata.local" -H "Content-Type: application/json" \
  http://192.168.1.12:32602/boats -d '{"name":"Bribón","boatClass":"ORC","sailNumber":"ESP-1234"}'
```

**Alternativa sin Ingress** (port-forward, ocupa la terminal):

```bash
kubectl port-forward -n regatas svc/session-service 8092:80 --address 0.0.0.0
curl http://192.168.1.12:8092/health
```

---

## `analysis-service` (regata-platform)

Tercer microservicio de `regata-platform`: API FastAPI que, para una sesión
dada, lee su telemetría de la InfluxDB propia del proyecto, detecta viradas y
trasluchadas (y su pérdida de VMG asociada) y persiste el resultado en
`session-service` (`POST /sessions/{sessionId}/maneuvers`). Se dispara **bajo
demanda** con un `curl` — todavía no hay `api-gateway` (Fase 4) que lo invoque
solo. Código fuente en `~/Documents/projects/regata-platform/` (repo
separado, privado). Manifiestos aquí bajo `apps/regata/analysis-service/`,
mismo namespace `regatas`.

Igual que `ingest-service` (y a diferencia de `session-service`), el contexto
de build es la **raíz** del monorepo `regata-platform`: depende de un paquete
local hermano (`libs/telemetry-schema`; ver ADR 0007 de ese repo):

```bash
cd ~/Documents/projects/regata-platform
docker build -f services/analysis-service/Dockerfile -t analysis-service:0.1.0 .
docker save analysis-service:0.1.0 | sudo k3s ctr images import -
```

### El único paso que nunca va al repo: el `Secret`

`configmap.yaml` lleva `INFLUXDB_URL`/`INFLUXDB_ORG`/`INFLUXDB_BUCKET`/
`SESSION_SERVICE_URL` (no sensibles). El token de InfluxDB sí lo es — mismo
valor que el de `ingest-service`, pero en su **propio** `Secret`: cada
`Application` de este repo lleva el suyo, para que sus manifiestos no
dependan de los de otra:

```bash
export KUBECONFIG=$HOME/.kube/config
kubectl create secret generic regata-analysis-service-influx \
  --from-literal=INFLUXDB_TOKEN=<el-token-de-regata-platform/.env> \
  -n regatas
```

Si el namespace `regatas` aún no existe, créalo antes o espera a que Argo lo
cree (`CreateNamespace=true`) y repite el `kubectl create secret`.

### Dar de alta la Application (solo la primera vez)

```bash
kubectl apply -f argocd/regata-analysis-service-app.yaml
```

### Probar

```bash
kubectl get application regata-analysis-service -n argocd     # Synced / Healthy
```

**Vía Ingress** (mismo NodePort 32602, enrutado por host) — necesita un
`session_id` ya ingerido (ver `ingest-service` más arriba):

```bash
curl -X POST -H "Host: analysis.regata.local" \
  http://192.168.1.12:32602/analyze/<session_id>
```

**Alternativa sin Ingress** (port-forward, ocupa la terminal):

```bash
kubectl port-forward -n regatas svc/analysis-service 8093:80 --address 0.0.0.0
curl -X POST http://192.168.1.12:8093/analyze/<session_id>
```

---

## `api-gateway` (regata-platform)

Cuarto microservicio de `regata-platform`, y el primero que solo **agrega**:
combina metadatos de `session-service`, telemetría de InfluxDB y análisis de
`analysis-service` (disparándolo bajo demanda si una sesión aún no tiene
maniobras, ADR 0003) en una sola respuesta para el frontend
(`GET /sessions`, `GET /sessions/{id}`). Código fuente en
`~/Documents/projects/regata-platform/` (repo separado, privado). Manifiestos
aquí bajo `apps/regata/api-gateway/`, mismo namespace `regatas`.

Igual que `session-service` (y a diferencia de `ingest-service`/
`analysis-service`), el contexto de build es la **propia carpeta** del
servicio: proyecto Maven autocontenido, sin paquetes hermanos del monorepo
(ADR 0007 de ese repo no aplica):

```bash
cd ~/Documents/projects/regata-platform/services/api-gateway
docker build -t api-gateway:0.1.0 .
docker save api-gateway:0.1.0 | sudo k3s ctr images import -
```

### El único paso que nunca va al repo: el `Secret`

`configmap.yaml` lleva `SESSION_SERVICE_URL`/`ANALYSIS_SERVICE_URL`/
`INFLUXDB_URL`/`INFLUXDB_ORG`/`INFLUXDB_BUCKET` (no sensibles). El token de
InfluxDB sí lo es — mismo valor que el de `ingest-service`/`analysis-service`,
pero en su **propio** `Secret` (cada `Application` de este repo lleva el suyo):

```bash
export KUBECONFIG=$HOME/.kube/config
kubectl create secret generic regata-api-gateway-influx \
  --from-literal=INFLUXDB_TOKEN=<el-token-de-regata-platform/.env> \
  -n regatas
```

Si el namespace `regatas` aún no existe, créalo antes o espera a que Argo lo
cree (`CreateNamespace=true`) y repite el `kubectl create secret`.

### Dar de alta la Application (solo la primera vez)

```bash
kubectl apply -f argocd/regata-api-gateway-app.yaml
```

### Probar

```bash
kubectl get application regata-api-gateway -n argocd     # Synced / Healthy
```

**Vía Ingress** (mismo NodePort 32602, enrutado por host):

```bash
curl -H "Host: gateway.regata.local" http://192.168.1.12:32602/sessions
curl -H "Host: gateway.regata.local" http://192.168.1.12:32602/sessions/<session_id>
```

**Alternativa sin Ingress** (port-forward, ocupa la terminal):

```bash
kubectl port-forward -n regatas svc/api-gateway 8094:80 --address 0.0.0.0
curl http://192.168.1.12:8094/sessions
```

---

## `frontend` (regata-platform)

Quinto componente de `regata-platform`, y el único que no es un microservicio
backend: React + MapLibre GL (mapa, replay de la traza) + uPlot (gráfica de
SOG/TWA sincronizada con el mapa por hover), servido como build estático por
Nginx. Lista sesiones y muestra el detalle de una (llama solo a
`api-gateway`, nunca a los otros tres servicios directamente). Código fuente
en `~/Documents/projects/regata-platform/` (repo separado, privado).
Manifiestos aquí bajo `apps/regata/frontend/`, mismo namespace `regatas`.

`VITE_API_BASE_URL` se hornea en el **build** (Vite no soporta configuración
en tiempo de ejecución): la imagen se construye ya apuntando al Ingress del
`api-gateway` de este mismo cluster.

```bash
cd ~/Documents/projects/regata-platform/services/frontend
docker build -t frontend:0.1.0 .
docker save frontend:0.1.0 | sudo k3s ctr images import -
```

Sin `Secret`: no hay nada sensible, todo lo que lleva es público (la URL del
gateway, que ya es un Ingress del propio cluster).

### Dar de alta la Application (solo la primera vez)

```bash
kubectl apply -f argocd/regata-frontend-app.yaml
```

### Probar

```bash
kubectl get application regata-frontend -n argocd     # Synced / Healthy
```

**Vía Ingress** (mismo NodePort 32602, enrutado por host) — añade
`192.168.1.12 frontend.regata.local gateway.regata.local` al `hosts` del
portátil (el frontend llama al gateway desde el navegador, así que ambos
hosts tienen que resolver) y abre:

```
http://frontend.regata.local:32602
```

**Alternativa sin Ingress** (port-forward, ocupa la terminal — pero entonces
`VITE_API_BASE_URL` horneado en el build, que apunta a
`gateway.regata.local:32602`, no lo alcanza el navegador salvo que también
resuelva ese host; más simple probar por Ingress con ambos hosts en `hosts`):

```bash
kubectl port-forward -n regatas svc/frontend 8096:80 --address 0.0.0.0
```
