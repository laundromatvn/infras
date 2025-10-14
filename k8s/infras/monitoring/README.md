# Kubernetes Monitoring with Prometheus, Grafana, and Loki

This document provides a step-by-step guide to setting up a monitoring system for a Kubernetes cluster using Prometheus, Grafana, and Loki. This setup is configured to use ephemeral storage, meaning that monitoring data will be lost when pods restart.

## Prerequisites

*   `kubectl` installed and configured to connect to your Kubernetes cluster.
*   `helm` version 3 or higher installed.

## Installation Steps

### 1. Add Helm Repositories

First, we need to add the Helm repositories for Prometheus and Grafana.

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

### 2. Install kube-prometheus-stack

Next, we will install the `kube-prometheus-stack`, which includes Prometheus for metrics collection and Grafana for visualization. We will disable persistent storage for all components.

```bash
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.prometheusSpec.storageSpec.emptyDir.sizeLimit=10Gi \
  --set prometheus.prometheusSpec.storageSpec.emptyDir.medium="" \
  --set grafana.persistence.enabled=false \
  --set alertmanager.persistence.enabled=false
```

### 3. Install Loki and Promtail

Now, we will install the `loki-stack`, which includes Loki for log aggregation and Promtail for log collection. We will disable persistent storage for Loki and configure Promtail to scrape logs from specific applications in the `lms` namespace.

```bash
helm install loki grafana/loki-stack \
  --namespace monitoring \
  --set loki.persistence.enabled=false \
  --set promtail.enabled=true \
  --set "promtail.config.snippets.pipelineStages[0].docker.labels.app.regex"="lms-backend|lms-mqtt-consumer" \
  --set "promtail.config.snippets.pipelineStages[0].docker.labels.namespace"="lms"
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

## Configuration

### Add Loki Data Source to Grafana

1.  Navigate to the Grafana UI in your browser.
2.  In the left-hand menu, go to the "Connections" section and click on "Data Sources".
3.  Click on "Add new data source" and select "Loki".
4.  In the "URL" field, enter `http://loki:3100`.
5.  Click "Save & test".

## Verification

### Verify Prometheus Targets

1.  Access the Prometheus UI.
2.  Navigate to the "Status" -> "Targets" page.
3.  Verify that all targets are in the "UP" state.

### Verify Log Scraping

1.  Access the Grafana UI.
2.  In the left-hand menu, click on the "Explore" icon.
3.  Select the "Loki" data source from the dropdown menu.
4.  Use the "Log browser" to query for your logs. For example, to see all logs from the `lms` namespace, use the query `{namespace="lms"}`.

## Troubleshooting

### Loki Data Source Connection Issues

If you are unable to connect to the Loki data source in Grafana, and you are seeing errors in the Grafana logs such as `dial tcp 10.96.183.184:80: i/o timeout`, it means that Grafana is trying to connect to Loki on the wrong port.

To resolve this, you can provision the Loki data source automatically by creating a configmap and updating the Grafana deployment.

1.  **Create a `loki-datasource.yaml` file with the following content:**

    ```yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: grafana-datasources-loki
      namespace: monitoring
      labels:
        grafana_datasource: "1"
    data:
      loki-datasource.yaml: |-
        apiVersion: 1
        datasources:
        - name: Loki
          type: loki
          url: http://loki:3100
          access: proxy
          isDefault: false
          jsonData:
            maxLines: 1000
    ```

2.  **Apply the configmap to your cluster:**

    ```bash
    kubectl apply -f loki-datasource.yaml
    ```

3.  **Update the Grafana deployment to mount the new configmap.**

    First, get the current Grafana deployment and save it to a file:

    ```bash
    kubectl get deployment -n monitoring prometheus-grafana -o yaml > grafana-deployment.yaml
    ```

    Then, edit the `grafana-deployment.yaml` file and add the following `volume` and `volumeMount` to the deployment spec:

    Under `spec.template.spec.volumes`, add:

    ```yaml
          - configMap:
              defaultMode: 420
              name: grafana-datasources-loki
            name: loki-datasource
    ```

    Under `spec.template.spec.containers.volumeMounts` for the `grafana` container, add:

    ```yaml
            - mountPath: /etc/grafana/provisioning/datasources/loki-datasource.yaml
              name: loki-datasource
              subPath: loki-datasource.yaml
    ```

4.  **Apply the updated deployment:**

    ```bash
    kubectl apply -f grafana-deployment.yaml
    ```

    This will restart the Grafana pod, and the Loki data source should be automatically provisioned.


