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
