# Kubernetes Observability Platform - Command Reference

This document contains the major commands used throughout the project, along with the purpose of each command and why it was used.

---

# 1. Minikube & Cluster Verification

## Check cluster status

```bash
minikube status
```

Purpose:
- Verifies Minikube cluster is running.

---

## View cluster nodes

```bash
kubectl get nodes
```

Purpose:
- Displays available Kubernetes nodes.

---

# 2. Application Deployment

## Deploy application

```bash
kubectl apply -f app.yaml
```

Purpose:
- Creates or updates Deployment resources.

---

## View pods

```bash
kubectl get pods -n observability
```

Purpose:
- Displays running application pods.

---

## Watch pod changes live

```bash
kubectl get pods -n observability -w
```

Purpose:
- Monitor pod creation, deletion, and rollout.

---

## Describe pod

```bash
kubectl describe pod <pod-name> -n observability
```

Purpose:
- Troubleshooting pod events and status.

---

## View pod logs

```bash
kubectl logs <pod-name> -n observability
```

Purpose:
- Check application output and errors.

---

# 3. Docker Commands

## Build image

```bash
docker build -t payment-service:v2 .
```

Purpose:
- Builds Flask application image.

---

## Load image into Minikube

```bash
minikube image load payment-service:v2
```

Purpose:
- Makes local Docker image available inside Minikube.

---

# 4. Kubernetes Service

## Create service

```bash
kubectl apply -f service.yaml
```

Purpose:
- Exposes deployment internally.

---

## List services

```bash
kubectl get svc -n observability
```

Purpose:
- Verify service creation and ports.

---

## Export service YAML

```bash
kubectl get svc payment-service -n observability -o yaml > service.yaml
```

Purpose:
- Save service configuration to file.

---

# 5. Prometheus & Grafana Installation

## Add Helm repository

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```

Purpose:
- Adds Prometheus chart repository.

---

## Update repositories

```bash
helm repo update
```

Purpose:
- Downloads latest chart metadata.

---

## Install Prometheus Stack

```bash
helm install monitoring prometheus-community/kube-prometheus-stack \
-n monitoring \
--create-namespace
```

Purpose:
- Installs:
  - Prometheus
  - Grafana
  - Alertmanager
  - Node Exporter

---

## View monitoring pods

```bash
kubectl get pods -n monitoring
```

Purpose:
- Verify installation success.

---

# 6. Grafana Access

## Port Forward Grafana

```bash
kubectl port-forward svc/monitoring-grafana 3000:80 -n monitoring
```

Purpose:
- Access Grafana locally.

URL:

```text
http://localhost:3000
```

---

# 7. ServiceMonitor

## Deploy ServiceMonitor

```bash
kubectl apply -f servicemonitor.yaml
```

Purpose:
- Instruct Prometheus to scrape custom metrics.

---

## Verify ServiceMonitor

```bash
kubectl describe servicemonitor payment-service-monitor -n monitoring
```

Purpose:
- Confirm ServiceMonitor configuration.

---

# 8. Metrics Verification

## Port Forward Application

```bash
kubectl port-forward svc/payment-service 8082:80 -n observability
```

Purpose:
- Access application locally.

---

## Check metrics endpoint

```bash
curl http://localhost:8082/metrics
```

Purpose:
- Verify Prometheus metrics output.

---

## Check response headers

```bash
curl -I http://localhost:8082/metrics
```

Purpose:
- Verify correct Prometheus content type.

---

# 9. Loki Installation

## Add Grafana Repository

```bash
helm repo add grafana https://grafana.github.io/helm-charts
```

Purpose:
- Adds Loki and Alloy charts.

---

## Install Loki

```bash
helm install loki grafana/loki \
-n logging \
--create-namespace
```

Purpose:
- Installs Loki log aggregation system.

---

## View Loki resources

```bash
kubectl get all -n logging
```

Purpose:
- Verify Loki installation.

---

# 10. Alloy Installation

## Install Alloy

```bash
helm install alloy grafana/alloy \
-n logging
```

Purpose:
- Collect logs from Kubernetes.

---

## View Alloy logs

```bash
kubectl logs -n logging daemonset/alloy
```

Purpose:
- Verify Alloy operation.

---

## Export Alloy ConfigMap

```bash
kubectl get configmap alloy -n logging -o yaml
```

Purpose:
- Inspect Alloy configuration.

---

## Apply custom Alloy config

```bash
kubectl apply -f alloy.yaml
```

Purpose:
- Configure log forwarding to Loki.

---

# 11. Loki Verification

## Check Loki readiness

```bash
kubectl exec -it -n monitoring deployment/monitoring-grafana -- \
wget -qO- http://loki.logging.svc.cluster.local:3100/ready
```

Purpose:
- Verify Grafana can reach Loki.

---

## Port Forward Loki

```bash
kubectl port-forward svc/loki 3100:3100 -n logging
```

Purpose:
- Access Loki locally for testing.

---

# 12. Metrics Server

## Enable Metrics Server

```bash
minikube addons enable metrics-server
```

Purpose:
- Provides CPU and memory metrics.

---

## View node metrics

```bash
kubectl top nodes
```

Purpose:
- Check node resource consumption.

---

## View pod metrics

```bash
kubectl top pods -n observability
```

Purpose:
- Check pod CPU and memory usage.

---

# 13. Horizontal Pod Autoscaler

## Create HPA

```bash
kubectl autoscale deployment payment-service \
--cpu-percent=50 \
--min=2 \
--max=6 \
-n observability
```

Purpose:
- Automatically scales application pods.

---

## View HPA

```bash
kubectl get hpa -n observability
```

Purpose:
- Check scaling status.

---

## Describe HPA

```bash
kubectl describe hpa payment-service -n observability
```

Purpose:
- Troubleshoot autoscaling issues.

---

# 14. Deployment Debugging

## View deployment YAML

```bash
kubectl get deployment payment-service \
-n observability -o yaml
```

Purpose:
- Inspect deployment configuration.

---

## Restart deployment

```bash
kubectl rollout restart deployment/payment-service -n observability
```

Purpose:
- Reload updated container image/configuration.

---

## View ReplicaSets

```bash
kubectl get rs -n observability
```

Purpose:
- Track deployment revisions.

---

# 15. General Troubleshooting

## Get all resources

```bash
kubectl get all -A
```

Purpose:
- Cluster-wide resource overview.

---

## Describe any resource

```bash
kubectl describe <resource> <name>
```

Purpose:
- Detailed debugging information.

---

## Delete resource

```bash
kubectl delete -f <file.yaml>
```

Purpose:
- Remove Kubernetes resources.

---

# Final Outcome

Using the commands above we successfully implemented:

✅ Dockerized Flask Application

✅ Kubernetes Deployment & Service

✅ Prometheus Metrics Collection

✅ Grafana Dashboards

✅ ServiceMonitor

✅ Loki Centralized Logging

✅ Alloy Log Collection

✅ Metrics Server

✅ Horizontal Pod Autoscaling

✅ End-to-End Observability Platform
