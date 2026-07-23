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
kubectl port-forward -n metrics-api svc/metrics-api 8090:80 --address 0.0.0.0
curl http://192.168.1.12:8090/api/metrics/resumen
```
