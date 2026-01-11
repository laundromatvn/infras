# Overview

Postgres <- Exporter -> Prometheus -> Grafana

# Guideline

## 1. Create account for postgres exporter
```
CREATE USER postgres_exporter WITH PASSWORD 'STRONG_PASSWORD';
GRANT CONNECT ON DATABASE postgres TO postgres_exporter;
GRANT pg_monitor TO postgres_exporter;
```

## 2. Secret

```secret.yml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-exporter-secret
  namespace: database
type: Opaque
stringData:
  DATA_SOURCE_NAME: |
    postgresql://postgres_exporter:STRONG_PASSWORD@postgres-clusterip.database.svc.cluster.local:5432/postgres?sslmode=disable
```

```
kubectl apply -f prometheus-postgres-exporter.secret.yaml
```

## 3. Deploy

```values.yml
replicaCount: 1

image:
  registry: quay.io
  repository: prometheuscommunity/postgres-exporter
  tag: v0.15.0
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 9187
  name: metrics

# Datasource configuration
config:
  datasource:
    host: "postgres-clusterip.database.svc.cluster.local"
    user: "postgres_exporter"
    pass: "STRONG_PASSWORD"
    port: "5432"
    database: "postgres"
    sslmode: "disable"

  # Metrics collectors configuration
  exporter:
    default_metrics:
      enabled: true

serviceMonitor:
  enabled: true
  namespace: database
  interval: 30s
  scrapeTimeout: 10s
  labels:
    release: prometheus # VERY IMPORTANT for kube-prometheus-stack

extraEnvs:
  - name: DATA_SOURCE_NAME
    valueFrom:
      secretKeyRef:
        name: postgres-exporter-secret
        key: DATA_SOURCE_NAME

resources:
  requests:
    cpu: 50m
    memory: 64Mi
  limits:
    cpu: 200m
    memory: 256Mi
```

Apply the below command

```
helm upgrade --install postgres-exporter \
  prometheus-community/prometheus-postgres-exporter \
  -n database \
  -f prometheus-postgres-exporter-values.yaml
```
