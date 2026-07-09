# TD noté Kubernetes — Mini Todo

## Pré-requis

Docker Desktop + k3d (`winget install k3d.k3d`) + kubectl, tout check.

```bash
k3d cluster create td-k8s --servers 1 --agents 2 --port "8080:80@loadbalancer" --k3s-arg "--disable=traefik@server:0"

docker pull stephanparichon/epsi-k8s-bff:1.0
docker pull stephanparichon/epsi-k8s-front:1.0
docker pull stephanparichon/epsi-k8s-front:2.0
k3d image import stephanparichon/epsi-k8s-bff:1.0 stephanparichon/epsi-k8s-front:1.0 stephanparichon/epsi-k8s-front:2.0 --cluster td-k8s
```

`kubectl get nodes` → 3 nodes Ready. Smoke test nginx OK sur localhost:8080.

## Étape 1 — Secret

Fait en impératif (pas de YAML committé, sinon le token traîne en clair/base64 dans le repo public) :

```bash
kubectl create secret generic backend-secret --from-literal=admin-token=s3cr3t-token-td
```

## Étape 2 — ConfigMap (`manifests/configmap.yaml`)

Le Service backend fourni s'appelle `backend`, port `3000`, même namespace → CoreDNS résout juste `backend`.

```yaml
data:
  BACKEND_HOST: "backend"
  BACKEND_PORT: "3000"
```

## Étape 3 — Deployment backend (`manifests/deployment-back.yaml`)

- 1 replica, labels `app: backend` (cohérent avec le selector de `service-back.yaml`)
- `ADMIN_TOKEN` injecté depuis le Secret (`secretKeyRef`)
- readiness + liveness sur `GET /health:3000` → pas de trafic tant que le pod n'est pas prêt, redémarre seul s'il crash

`kubectl get pods -l app=backend` → 1/1 Running.

## Étape 4 — Deployment frontend (`manifests/deployment-front.yaml`)

- 2 replicas, `envFrom` la ConfigMap `frontend-config`
- readiness sur `GET /health:80`, initialDelay 2s
- stratégie de rollout :

```yaml
strategy:
  rollingUpdate:
    maxUnavailable: 0
    maxSurge: 1
```

→ jamais moins de 2 pods dispo pendant une update = zéro coupure.

(Rappel du sujet : nginx résout le DNS du backend au démarrage, pas à chaque requête — donc éviter que le backend redémarre pour rien une fois le front up.)

## Étape 5 — Services

- Backend : fourni (`service-back.yaml`), ClusterIP.
- Frontend : écrit (`service-front.yaml`), LoadBalancer pour sortir sur `localhost:8080`.

## Étape 6 — Appliquer et tester

```bash
kubectl apply -f manifests/configmap.yaml
kubectl apply -f manifests/deployment-back.yaml
kubectl apply -f manifests/service-back.yaml
kubectl apply -f manifests/deployment-front.yaml
kubectl apply -f manifests/service-front.yaml
```

Tout passe Ready. App testée sur `http://localhost:8080`, todos ajoutées/supprimées, nom du pod backend affiché en haut.

📸 `screenshots/01-app-v1.png`

## Étape 7 — Rolling update v1 → v2

```bash
kubectl get pods -l app=frontend -w   # dans un autre terminal
# édition de deployment-front.yaml : front:1.0 -> front:2.0
kubectl apply -f manifests/deployment-front.yaml
kubectl rollout status deployment/frontend
```

Rollout terminé sans jamais passer sous 2 pods dispo. Le todo créé en v1 est toujours là après (seuls les pods frontend changent, le backend n'a pas bougé).

📸 avant `screenshots/01-app-v1.png` / après `screenshots/02-app-v2.png`

## Étape 8 — Déploiement raté & rollback

Édition de `deployment-front.yaml` avec un tag qui n'existe pas (`front:9.9`) :

```bash
kubectl apply -f manifests/deployment-front.yaml
kubectl get pods -l app=frontend
```

Le nouveau pod reste en `ImagePullBackOff`, jamais Ready. Comme `maxUnavailable: 0`, le rollout ne touche pas les 2 anciens pods : l'app continue de tourner normalement sur localhost:8080 pendant tout ce temps.

📸 `screenshots/03-deploiement-rate.png`

Retour en arrière :

```bash
kubectl rollout undo deployment/frontend
kubectl rollout status deployment/frontend
```

Rollback natif plutôt que réappliquer le YAML v2.0 : Kubernetes garde l'historique des ReplicaSet, donc `rollout undo` remonte instantanément l'ancien ReplicaSet déjà présent (pas besoin de re-pull l'image ni de retrouver le bon fichier). Le fichier `deployment-front.yaml` a été remis à `front:2.0` après coup pour matcher l'état réel.

📸 `screenshots/04-rollback.png`

## Étape 9 — Test du Secret avec curl

```bash
curl -s -o /dev/null -w "%{http_code}\n" -X POST http://localhost:8080/api/admin/clear
# 401 (pas de header)

curl -s -o /dev/null -w "%{http_code}\n" -X POST http://localhost:8080/api/admin/clear -H "X-Admin-Token: mauvais-token"
# 401

curl -s -X POST http://localhost:8080/api/admin/clear -H "X-Admin-Token: s3cr3t-token-td"
# 200 + {"status":"ok","cleared":1,...}
```

Todos bien vidées après (visible dans `04-rollback.png`, prise après ce test).
