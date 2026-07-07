 
# Kubernetes Logging Stack: Loki & Fluent Bit Installation Guide

## 📋 Table of Contents
- [Prerequisites](#prerequisites)
- [Loki Installation](#loki-installation)
- [Storage Configuration](#storage-configuration)
- [Troubleshooting Stuck PVCs](#troubleshooting-stuck-pvcs)
- [Verification](#verification)
- [Grafana Integration](#grafana-integration)
- [Fluent Bit Installation](#fluent-bit-installation)
- [Fluent Bit Pipeline Explained](#fluent-bit-pipeline-explained)
- [Configuration Reference](#configuration-reference)

## 🚀 Prerequisites
- Kubernetes cluster (v1.19+)
- `kubectl` configured
- Basic understanding of Kubernetes concepts

## 📦 Loki Installation

### Deploy Loki using sample application YAML:
```bash
kubectl apply -f samples/addons/loki.yaml
```

> **Note**: This deploys Loki in the `istio-system` namespace by default.

## 💾 Storage Configuration

### For Bare-Metal / VM Clusters

**1. Install local-path provisioner:**
```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
```

**2. Set as default StorageClass:**
```bash
kubectl patch storageclass local-path -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

### For Cloud Providers
Ensure your cloud provider's StorageClass is set as default:
```bash
kubectl patch storageclass <cloud-provider-storage-class> -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

## 🔧 Troubleshooting Stuck PVCs

If Loki pod is stuck in `Pending` state due to PVC issues:

### Step 1: Delete stuck PVC
```bash
kubectl delete pvc storage-loki-0 -n istio-system
```

### Step 2: Delete Loki pod (forces recreation)
```bash
kubectl delete pod loki-0 -n istio-system
```
> **Note**: StatefulSet will automatically recreate the pod.

### Step 3: Watch provisioning status
```bash
kubectl get pvc -n istio-system -w
```

## ✅ Verification

### Check Loki Readiness:
```bash
curl http://<loki-service-ip>:3100/ready
```
Expected response: `ready`

### Check Metrics Endpoint:
```bash
curl http://<loki-service-ip>:3100/metrics
```

## 📊 Grafana Integration

### Configure Loki Data Source in Grafana:

1. Navigate to **Settings** → **Data Sources**
2. Click **Add data source**
3. Select **Loki**
4. Configure URL:
   - **URL**: `http://loki:3100`
   - (If in same namespace) or `http://loki.istio-system.svc.cluster.local:3100`
5. Click **Save & Test**
6. Verify connection success message

## 🔄 Fluent Bit Installation (Log Shipper)

### 1. Service Account & RBAC

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluent-bit
  namespace: logging
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: fluent-bit
rules:
  - apiGroups: [""]
    resources:
      - pods
      - namespaces
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: fluent-bit
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: fluent-bit
subjects:
  - kind: ServiceAccount
    name: fluent-bit
    namespace: logging
```

### 2. ConfigMap Configuration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: logging
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush        5
        Daemon       Off
        Log_Level    info
        Parsers_File /fluent-bit/etc/parsers.conf
    
    [INPUT]
        Name tail
        Path /var/log/containers/*.log
        Parser cri
        Tag kube.*

    [FILTER]
        Name kubernetes
        Match kube.*
        kube_url            https://kubernetes.default.svc:443
        kube_ca_file        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        kube_token_file     /var/run/secrets/kubernetes.io/serviceaccount/token
        merge_log           on
        merge_log_trim      on
        keep_log            off
        k8s-logging.parser  on
        k8s-logging.exclude off
        labels              on
        annotations         off

    [FILTER]
        Name         record_modifier
        Match        kube.*
        record       cluster_name ${CLUSTER_NAME}
        record       environment production
    
    [OUTPUT]
        Name loki
        Match *
        Host loki.istio-system.svc.cluster.local
        Port 3100
        Labels job=fluentbit,namespace=$kubernetes['namespace_name'],pod=$kubernetes['pod_name'],service=$kubernetes['labels']['app']
        Line_Format json
```

### 3. DaemonSet Deployment

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: logging
spec:
  selector:
    matchLabels:
      app: fluent-bit
  template:
    metadata:
      labels:
        app: fluent-bit
    spec:
      serviceAccountName: fluent-bit
      containers:
        - name: fluent-bit
          image: cr.fluentbit.io/fluent/fluent-bit:2.2
          volumeMounts:
            - name: varlog
              mountPath: /var/log
            - name: config
              mountPath: /fluent-bit/etc/
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
        - name: config
          configMap:
            name: fluent-bit-config
```

### Apply All Fluent Bit Resources:
```bash
# Create namespace
kubectl create namespace logging

# Apply configurations
kubectl apply -f fluent-bit-rbac.yaml
kubectl apply -f fluent-bit-configmap.yaml
kubectl apply -f fluent-bit-daemonset.yaml
```

## 🔄 Fluent Bit Pipeline Explained

### Data Flow Architecture:
```
┌──────────────┐
│   INPUT      │
│ (Log Source) │
└──────┬───────┘
       ▼
┌──────────────┐
│   PARSER     │
│ (Structure   │
│  raw logs)   │
└──────┬───────┘
       ▼
┌──────────────┐
│ FILTER 1     │
│ (Kubernetes  │
│ metadata add)│
└──────┬───────┘
       ▼
┌──────────────┐
│ FILTER 2     │
│ (Grep/Drop/  │
│ Modify logs) │
└──────┬───────┘
       ▼
┌──────────────┐
│ FILTER N     │
│ (Extra rules)│
└──────┬───────┘
       ▼
┌──────────────┐
│   OUTPUT     │
│ (Loki/ES/etc)│
└──────────────┘
```

## ⚙️ Configuration Reference

### [SERVICE] Section Parameters

| Parameter | Value | Description |
|-----------|-------|-------------|
| `Flush` | 5 | Sends logs to output every 5 seconds (Lower = faster delivery, Higher = better performance) |
| `Daemon` | Off | Runs in foreground (Off = Kubernetes mode, On = background process) |
| `Log_Level` | info | Controls internal logs (error, warn, info, debug) |
| `Parsers_File` | /fluent-bit/etc/parsers.conf | Loads parsing rules (CRI, JSON, etc.) |

### Monitoring & Control API Server

| Parameter | Description |
|-----------|-------------|
| `http_server` | Switches on integrated internal web server for health metrics |
| `http_listen` | Binds telemetry server API to specific local interface address |
| `http_port` | Designates precise TCP port allocation for server endpoint |
| `hot_reload` | Enables configuration updates in real-time using SIGHUP process signals |

## 📝 Summary

| Section | Purpose |
|---------|---------|
| `[SERVICE]` | Fluent Bit engine settings |
| `[INPUT]` | Log collection configuration |
| `[FILTER]` | Log processing and enrichment rules |
| `[OUTPUT]` | Log shipping destination configuration |

## 🎯 Key Takeaways

- **Loki** serves as the log storage and query system
- **Fluent Bit** acts as the lightweight log shipper
- **Filters** in Fluent Bit can: modify fields, add metadata, remove data, or drop entire records
- **StatefulSet** handles Loki persistence and recovery
- Each log record flows through the pipeline **sequentially**

 
 
