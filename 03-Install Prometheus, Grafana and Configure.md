# Istio Monitoring Setup Guide

## Overview

This guide provides step-by-step instructions for setting up monitoring in an Istio service mesh using Prometheus and Grafana. It covers installation, configuration, scraping validation, and dashboard setup.

---

## 1. Installing Prometheus

Istio provides a basic sample installation to quickly get Prometheus up and running:

```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.30/samples/addons/prometheus.yaml
```

> **Note:** This deployment is intended for demonstration purposes only and is not tuned for performance or security.

---

## 2. Understanding Prometheus Configuration

### How It Works

In an Istio mesh, each component exposes an endpoint that emits metrics. Prometheus scrapes these endpoints and collects the results. Configuration controls:

- Which endpoints to query
- Port and path to query
- TLS settings
- Scrape intervals and timeouts

### Components to Scrape

To gather metrics for the entire mesh, configure Prometheus to scrape:

- **Control Plane** (`istiod` deployment)
- **Ingress and Egress Gateways**
- **Envoy Sidecar** (each pod)
- **User Applications** (if they expose Prometheus metrics)

---

## 3. Configuration Modes

Istio offers two modes of operation to simplify metrics configuration.

### Option 1: Metrics Merging (Default)

```bash
kubectl -n istio-system get configmap istio -o jsonpath='{.data.mesh}' | grep enablePrometheusMerge
```

**Expected output:**
```
enablePrometheusMerge: true
```

This option enables Istio to control scraping entirely through `prometheus.io` annotations, allowing it to work out of the box with standard configurations.

To disable this feature (if needed):
```bash
--set meshConfig.enablePrometheusMerge=false
```

### Option 2: Manual Configuration

For custom scraping requirements, you can manually configure Prometheus using the `prometheus.yml` configuration file.

---

## 4. Complete prometheus.yaml Configuration

Below is the complete configuration file including `istiod`, Envoy, and sidecar metrics:

### Global Configuration

```yaml
global:
  evaluation_interval: 1m
  scrape_interval: 15s
  scrape_timeout: 10s
```

### Scrape Jobs

#### Kubernetes API Servers

```yaml
- job_name: kubernetes-api-servers
  bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
  kubernetes_sd_configs:
    - role: endpoints
  relabel_configs:
    - action: keep
      regex: default;kubernetes;https
      source_labels:
        - __meta_kubernetes_namespace
        - __meta_kubernetes_service_name
        - __meta_kubernetes_endpoint_port_name
  scheme: https
  tls_config:
    ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
```

#### Kubernetes Nodes

```yaml
- job_name: kubernetes-nodes
  bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
  kubernetes_sd_configs:
    - role: node
  relabel_configs:
    - action: labelmap
      regex: __meta_kubernetes_node_label_(.+)
  scheme: https
  tls_config:
    ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    insecure_skip_verify: true
```

#### Kubernetes Nodes (cAdvisor)

```yaml
- job_name: kubernetes-nodes-cadvisor
  bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
  kubernetes_sd_configs:
    - role: node
  metrics_path: /metrics/cadvisor
  relabel_configs:
    - action: labelmap
      regex: __meta_kubernetes_node_label_(.+)
    - source_labels: [__metrics_path__]
      target_label: metrics_path
  scheme: https
  tls_config:
    ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    insecure_skip_verify: true
```

#### Kubernetes Pods (Standard)

```yaml
- job_name: kubernetes-pods
  honor_labels: true
  kubernetes_sd_configs:
    - role: pod
  relabel_configs:
    - action: keep
      regex: true
      source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_scrape
    - action: drop
      regex: true
      source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_scrape_slow
    - action: replace
      regex: (https?)
      source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_scheme
      target_label: __scheme__
    - action: replace
      regex: (.+)
      source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_path
      target_label: __metrics_path__
    - action: replace
      regex: (\d+);(([A-Fa-f0-9]{1,4}::?){1,7}[A-Fa-f0-9]{1,4})
      replacement: '[$2]:$1'
      source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_port
        - __meta_kubernetes_pod_ip
      target_label: __address__
    - action: replace
      regex: (\d+);((([0-9]+?)(\.|$)){4})
      replacement: $2:$1
      source_labels:
        - __meta_kubernetes_pod_annotation_prometheus_io_port
        - __meta_kubernetes_pod_ip
      target_label: __address__
    - action: labelmap
      regex: __meta_kubernetes_pod_annotation_prometheus_io_param_(.+)
      replacement: __param_$1
    - action: labelmap
      regex: __meta_kubernetes_pod_label_(.+)
    - action: replace
      source_labels:
        - __meta_kubernetes_namespace
      target_label: namespace
    - action: replace
      source_labels:
        - __meta_kubernetes_pod_name
      target_label: pod
    - action: drop
      regex: Pending|Succeeded|Failed|Completed
      source_labels:
        - __meta_kubernetes_pod_phase
    - action: replace
      source_labels:
        - __meta_kubernetes_pod_node_name
      target_label: node
```

#### Kubernetes Pods (Slow Scrape)

```yaml
- job_name: kubernetes-pods-slow
  honor_labels: true
  kubernetes_sd_configs:
    - role: pod
  scrape_interval: 5m
  scrape_timeout: 30s
  relabel_configs:
    # Same relabel configs as above
    # ... (full configuration included in original file)
```

#### Kubernetes Service Endpoints

```yaml
- job_name: kubernetes-service-endpoints
  honor_labels: true
  kubernetes_sd_configs:
    - role: endpoints
  relabel_configs:
    - action: keep
      regex: true
      source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_scrape
    - action: drop
      regex: true
      source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_scrape_slow
    - action: replace
      regex: (https?)
      source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_scheme
      target_label: __scheme__
    - action: replace
      regex: (.+)
      source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_path
      target_label: __metrics_path__
    - action: replace
      regex: (.+?)(?::\d+)?;(\d+)
      replacement: $1:$2
      source_labels:
        - __address__
        - __meta_kubernetes_service_annotation_prometheus_io_port
      target_label: __address__
    - action: labelmap
      regex: __meta_kubernetes_service_annotation_prometheus_io_param_(.+)
      replacement: __param_$1
    - action: labelmap
      regex: __meta_kubernetes_service_label_(.+)
    - action: replace
      source_labels:
        - __meta_kubernetes_namespace
      target_label: namespace
    - action: replace
      source_labels:
        - __meta_kubernetes_service_name
      target_label: service
    - action: replace
      source_labels:
        - __meta_kubernetes_pod_node_name
      target_label: node
```

#### Envoy Stats

```yaml
- job_name: envoy-stats
  metrics_path: /stats/prometheus
  kubernetes_sd_configs:
    - role: pod
  relabel_configs:
    - source_labels: [__meta_kubernetes_pod_container_port_name]
      action: keep
      regex: '.*-envoy-prom'
    - action: labelmap
      regex: __meta_kubernetes_pod_label_(.+)
    - source_labels: [__meta_kubernetes_namespace]
      action: replace
      target_label: namespace
    - source_labels: [__meta_kubernetes_pod_name]
      action: replace
      target_label: pod_name
```

#### Istiod

```yaml
- job_name: istiod
  kubernetes_sd_configs:
    - role: endpoints
      namespaces:
        names:
          - istio-system
  relabel_configs:
    - source_labels:
        - __meta_kubernetes_service_name
        - __meta_kubernetes_endpoint_port_name
      action: keep
      regex: istiod;http-monitoring
```

#### Prometheus Self-Monitoring

```yaml
- job_name: prometheus
  static_configs:
    - targets:
        - localhost:9090
```

#### Prometheus Pushgateway

```yaml
- job_name: prometheus-pushgateway
  honor_labels: true
  kubernetes_sd_configs:
    - role: service
  relabel_configs:
    - action: keep
      regex: pushgateway
      source_labels:
        - __meta_kubernetes_service_annotation_prometheus_io_probe
```

### Rule Files

```yaml
rule_files:
  - /etc/config/recording_rules.yml
  - /etc/config/alerting_rules.yml
  - /etc/config/rules
  - /etc/config/alerts
```

---

## 5. Validating Scraping

### Check Targets in Prometheus UI

1. Open the Prometheus UI
2. Navigate to **Status → Targets**

**You should see healthy targets:**
- ✅ `istiod` - Status: UP
- ✅ `envoy-stats` - Status: UP
- ✅ `kubernetes-pods` - Status: UP

### Test Metrics Immediately

In the Prometheus query bar, run:

```promql
istio_requests_total
```

**If working correctly:**
- You will see increasing values
- Labels like `source_service` and `destination_service` will be present

---

## 6. Useful PromQL Queries

### Request Rate by Service

```promql
sum(rate(istio_requests_total{reporter="destination"}[5m])) by (destination_service)
```

### Error Rate by Service

```promql
sum(rate(istio_requests_total{reporter="destination", response_code=~"5.."}[5m]))
by (destination_service)
/
sum(rate(istio_requests_total{reporter="destination"}[5m]))
by (destination_service)
```

### P99 Latency by Service

```promql
histogram_quantile(0.99,
  sum(rate(istio_request_duration_milliseconds_bucket{reporter="destination"}[5m]))
  by (le, destination_service)
)
```

### Request Rate Between Specific Services

```promql
sum(rate(istio_requests_total{
  reporter="destination",
  source_workload="productpage-v1",
  destination_workload="reviews-v1"
}[5m]))
```

### TCP Connection Rate

```promql
sum(rate(istio_tcp_connections_opened_total{reporter="destination"}[5m]))
by (destination_service)
```

---

## 7. Performance Optimization

To reduce data volume, increase the scrape interval:

```yaml
# In Prometheus scrape config
- job_name: 'envoy-stats'
  scrape_interval: 30s   # Increased from 15s to 30s
```

> **Effect:** Going from 15s to 30s halves the data volume.

---

## 8. Installing Grafana

Istio provides a basic sample installation for Grafana with pre-installed Istio dashboards:

```bash
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.30/samples/addons/grafana.yaml
```

> **Note:** This deployment is intended for demonstration purposes only and is not tuned for performance or security.

---

## 9. Configuring Grafana

### Add Prometheus as Data Source

1. Go to **Settings** (⚙️) → **Data Sources**
2. Click **Add data source**
3. Select **Prometheus**
4. Set the URL:

   **If Prometheus is in the same cluster:**
   ```
   http://prometheus-server.monitoring.svc.cluster.local:80
   ```
   
   **OR**
   
   ```
   http://prometheus-k8s.monitoring.svc:9090
   ```
   > (Depends on your installation)

5. Click **Save & Test**

**Expected result:** `Data source is working`

---

## 10. Importing Dashboards

### Dashboard Import Steps

1. In Grafana, click **+** → **Import**
2. Enter the dashboard ID
3. Select the Prometheus data source
4. Click **Import**

### Recommended Dashboards for Kubernetes and Istio

| Component | Dashboard ID | Description |
|-----------|--------------|-------------|
| Kubernetes Cluster | 315 | Node CPU, memory, pod status, network traffic |
| Node Exporter | 1860 | CPU per core, RAM, disk I/O, system load |
| Istio Mesh | 7639 | Istio service mesh overview |
| Istio Service Dashboard | 7636 | Service-level metrics |
| Prometheus Stats | 3662 | Prometheus self-monitoring |

### Additional Useful Dashboards

- **6417** - Kubernetes API / Cluster Monitoring
- **7249** - Kubernetes Cluster Monitoring (alternative)
- **8588** - Kubernetes API Server

---

## 11. Quick Start Summary

### Complete Setup Commands

```bash
# Install Prometheus
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.30/samples/addons/prometheus.yaml

# Install Grafana
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.30/samples/addons/grafana.yaml

# Verify scraping
# Access Prometheus UI: Status → Targets

# Query test
# In Prometheus: istio_requests_total

# Configure Grafana
# Add Prometheus data source
# Import dashboard IDs: 315, 1860, 7639, 7636, 3662
```

### Recommended Dashboard Stack

For a comprehensive monitoring solution with Kubernetes + Istio:

| Component | Dashboard ID |
|-----------|--------------|
| Kubernetes Cluster | 315 |
| Node Exporter | 1860 |
| Istio Mesh | 7639 |
| Istio Service Dashboard | 7636 |
| Prometheus Stats | 3662 |

---

## 12. Troubleshooting

### Common Issues

| Issue | Solution |
|-------|----------|
| Scrape targets down | Check network policies and service endpoints |
| No Istio metrics | Verify `enablePrometheusMerge: true` in mesh config |
| Data source not working | Confirm Prometheus service URL and port |
| Missing dashboards | Import dashboard IDs again |
| High data volume | Increase scrape intervals or filter targets |

---
 
