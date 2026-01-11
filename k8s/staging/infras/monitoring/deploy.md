# Deploy Monitoring for STAGING → PROD (Kubernetes)

## Mục tiêu
- Thu thập metrics và logs từ cluster STAGING
- Hiển thị toàn bộ trên Grafana ở cluster PROD
- Không deploy Grafana ở STAGING

---

## 1. Chuẩn bị

### Yêu cầu
- kubectl context trỏ về STAGING
- Helm v3+
- Network STAGING → PROD thông suốt

### Endpoint cần có
- PROMETHEUS_PROD_ENDPOINT
- LOKI_PROD_ENDPOINT

---

## 2. Add Helm repositories

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

---

## 3. Deploy Prometheus on Staging

```bash
helm upgrade --install prometheus \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  -f prometheus-values.yaml
```

Verify
```
kubectl -n monitoring get pods
```

---

## 4. Deploy Promtail on Staging

```bash
helm upgrade --install promtail \
  grafana/promtail \
  --namespace monitoring \
  -f promtail-values.yaml
```

Verify

```
kubectl -n monitoring logs -l app.kubernetes.io/name=promtail
```

---

## 5. Verify Grafana on Production

Metrics
```
sum(rate(container_cpu_usage_seconds_total{cluster="staging"}[5m]))
```

Logs
```
{cluster="staging", namespace="lms"}
```
