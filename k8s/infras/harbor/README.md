# Harbor on Kubernetes (Helm)

## Prereqs
- Namespace `harbor` created
- External Postgres available at `postgres-clusterip.database.svc.cluster.local:5432`
- External Redis available at `redis-clusterip.redis.svc.cluster.local:6379`
- TLS secret `harbor-tls` present in `harbor` namespace
- Ingress class `vngcloud` available

## One-time DB init
```bash
kubectl apply -f k8s/harbor/secrets.yaml
kubectl apply -f k8s/harbor/db-init-job.yaml
kubectl -n harbor logs job/harbor-db-init -f
```

## Install/Upgrade via Helm
```bash
helm repo add harbor https://helm.goharbor.io
helm repo update
helm upgrade --install harbor harbor/harbor \
  -n harbor \
  -f k8s/harbor/values.yaml
```

## Notes
- Admin user: `admin`; set password in `harborAdminPassword`.
- Access UI at `https://harbor.washgo247.com`.
- Update `values.yaml` if DB/Redis endpoints or passwords change.


