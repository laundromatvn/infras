# Kubernetes Monitoring with Prometheus, Grafana, and Loki

This document provides a step-by-step guide to setting up a monitoring system for a Kubernetes cluster using Prometheus, Grafana, and Loki. This setup is configured to use persistent storage.

## Prerequisites

*   `kubectl` installed and configured to connect to your Kubernetes cluster.
*   `helm` version 3 or higher installed.

## Installation Steps

### 1. Create Values Files

Create the following files with the content below:

**`prometheus-values.yaml`**
```yaml
# prometheus-values.yaml
grafana:
  enabled: true
  adminUser: admin
  adminPassword: admin
  persistence:
    enabled: true
    storageClassName: vngcloud-ssd-3000-delete
    accessModes:
      - ReadWriteOnce
    size: 3Gi
  # Optional: expose Grafana inside cluster only
  service:
    type: ClusterIP
    port: 80

prometheus:
  prometheusSpec:
    retention: 30d               # only 30d metrics
    scrapeInterval: 15s
    evaluationInterval: 30s
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: vngcloud-ssd-3000-delete
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 10Gi

alertmanager:
  enabled: false  # disable for now
```

**`loki-values.yaml`**
```yaml
# loki-values.yaml
persistence:
  enabled: true
  storageClassName: vngcloud-ssd-3000-delete
  accessModes:
    - ReadWriteOnce
  size: 5Gi

deploymentMode: SingleBinary
write:
  replicas: 0
read:
  replicas: 0
backend:
  replicas: 0

loki:
  commonConfig:
    replication_factor: 1
  storage:
    type: filesystem
  schemaConfig:
    configs:
      - from: "2020-10-24"
        store: boltdb-shipper
        object_store: filesystem
        schema: v11
        index:
          prefix: index_
          period: 24h
  compactor:
    retention_enabled: false
```

**`promtail-values.yaml`**
```yaml
# promtail-values.yaml
persistence:
  enabled: true
  storageClassName: vngcloud-ssd-3000-delete
  accessModes:
    - ReadWriteOnce
  size: 2Gi
daemonset:
  enabled: true

config:
  server:
    http_listen_port: 9080
  positions:
    filename: /tmp/positions.yaml
  clients:
    - url: http://loki.monitoring.svc.cluster.local:3100/loki/api/v1/push

  scrape_configs:
    - job_name: kubernetes-pods-lms
      pipeline_stages:
        - docker: {}
      kubernetes_sd_configs:
        - role: pod
      relabel_configs:
        # Only namespace "lms"
        - source_labels: [__meta_kubernetes_namespace]
          action: keep
          regex: lms
        # Only lms-backend or lms-mqtt-consumer pods
        - source_labels: [__meta_kubernetes_pod_name]
          action: keep
          regex: 'lms-backend.*|lms-mqtt-consumer.*'
        - source_labels: [__meta_kubernetes_pod_name]
          target_label: pod
        - source_labels: [__meta_kubernetes_namespace]
          target_label: namespace
        - source_labels: [__meta_kubernetes_container_name]
          target_label: container
```

### 2. Add Helm Repositories

First, we need to add the Helm repositories for Prometheus and Grafana.

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

### 3. Install kube-prometheus-stack

Next, we will install the `kube-prometheus-stack`, which includes Prometheus for metrics collection and Grafana for visualization.

```bash
helm upgrade --install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  -f prometheus-values.yaml
```

### 4. Install Loki and Promtail

Now, we will install the `loki-stack`, which includes Loki for log aggregation and Promtail for log collection.

```bash
helm upgrade --install loki grafana/loki-stack \
  --namespace monitoring \
  -f loki-values.yaml \
  -f promtail-values.yaml
```

## Accessing Grafana and Prometheus

### Accessing Grafana

1.  **Get the Grafana admin password:**

    ```bash
    kubectl --namespace monitoring get secrets prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 -d ; echo
    ```

2.  **Port-forward to the Grafana service:**

    ```bash
    export POD_NAME=$(kubectl --namespace monitoring get pod -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=prometheus" -oname)
    kubectl --namespace monitoring port-forward $POD_NAME 3000
    ```

3.  Open your browser and navigate to `http://localhost:3000`. Log in with the username `admin` and the password you retrieved.

### Accessing Prometheus

1.  **Port-forward to the Prometheus service:**

    ```bash
    export POD_NAME=$(kubectl --namespace monitoring get pod -l "app=prometheus,component=prometheus" -oname)
    kubectl --namespace monitoring port-forward $POD_NAME 9090
    ```

2.  Open your browser and navigate to `http://localhost:9090`.