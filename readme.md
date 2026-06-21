# Kubernetes Observability Platform 🚀

## Project Overview

This project demonstrates a complete cloud-native observability stack running on Kubernetes. The goal was to deploy a containerized application, collect metrics and logs, visualize them in Grafana, and configure autoscaling based on resource usage.

The project simulates how modern DevOps and SRE teams monitor applications in production.

---

# Problem Statement

When applications run in Kubernetes, engineers need answers to questions like:

- Is the application healthy?
- How many requests is it handling?
- What are the CPU and memory usages?
- What logs are being generated?
- How can logs from multiple pods be centralized?
- How can the application automatically scale when traffic increases?

Without observability tools, troubleshooting production issues becomes difficult.

---

# Objective

Build an end-to-end observability platform that provides:

- Metrics Monitoring
- Log Aggregation
- Dashboard Visualization
- Application Health Monitoring
- Kubernetes Autoscaling

---

# Architecture

```text
User
 │
 ▼
Payment Service (Flask)
 │
 ├── Metrics (/metrics)
 │         │
 │         ▼
 │    Prometheus
 │         │
 │         ▼
 │      Grafana
 │
 └── Logs
           │
           ▼
         Alloy
           │
           ▼
          Loki
           │
           ▼
        Grafana

Metrics Server
      │
      ▼
      HPA
      │
      ▼
Payment Service Pods
```

---

# Technologies Used

| Technology | Purpose |
|------------|----------|
| Docker | Containerization |
| Kubernetes | Container Orchestration |
| Flask | Sample Application |
| Prometheus | Metrics Collection |
| Grafana | Dashboards & Visualization |
| Loki | Log Storage |
| Alloy | Log Collection |
| Metrics Server | Resource Metrics |
| HPA | Horizontal Pod Autoscaling |
| kubectl | Kubernetes Management |
| Helm | Kubernetes Package Manager |

---

# Phase 1: Application Deployment

## Why?

Before monitoring anything, an application must be running inside Kubernetes.

## Flask Application

Endpoints:

- `/`
- `/health`
- `/metrics`

Custom metric:

```python
http_requests_total
```

Tracks total requests handled by the application.

---

# Dockerization

## Why?

Containers ensure the application runs consistently across environments.

Build Image:

```bash
docker build -t payment-service:v2 .
```

Load into Minikube:

```bash
minikube image load payment-service:v2
```

---

# Kubernetes Deployment

## Deployment

Purpose:

- Runs application pods
- Maintains desired replicas
- Performs rolling updates

Applied using:

```bash
kubectl apply -f app.yaml
```

Verify:

```bash
kubectl get pods -n observability
```

---

# Kubernetes Service

## Why?

Pods are temporary.

Services provide stable networking.

Created:

```yaml
type: ClusterIP
```

Applied:

```bash
kubectl apply -f service.yaml
```

Verify:

```bash
kubectl get svc -n observability
```

---

# Phase 2: Prometheus Monitoring

## Why Prometheus?

Prometheus scrapes metrics from applications and stores time-series data.

Examples:

- Request count
- CPU usage
- Memory usage
- Pod metrics

---

# Install Prometheus + Grafana

Added Helm Repo:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

Installed Stack:

```bash
helm install monitoring prometheus-community/kube-prometheus-stack \
-n monitoring \
--create-namespace
```

---

# ServiceMonitor

## Problem

Prometheus does not automatically know which applications to scrape.

## Solution

Create ServiceMonitor.

Purpose:

- Tells Prometheus where metrics exist
- Scrapes `/metrics` endpoint every 15 seconds

Applied:

```bash
kubectl apply -f servicemonitor.yaml
```

Verify:

```bash
kubectl describe servicemonitor payment-service-monitor -n monitoring
```

---

# Metrics Endpoint Fix

Issue encountered:

```text
404 Not Found
```

Reason:

Application did not expose Prometheus metrics correctly.

Solution:

```python
from prometheus_client import CONTENT_TYPE_LATEST
from flask import Response
```

Return:

```python
Response(
    generate_latest(),
    mimetype=CONTENT_TYPE_LATEST
)
```

Verification:

```bash
curl http://localhost:8082/metrics
```

Expected:

```text
python_gc_objects_collected_total
http_requests_total
process_cpu_seconds_total
```

---

# Grafana

Access:

```bash
kubectl port-forward svc/monitoring-grafana 3000:80 -n monitoring
```

Open:

```text
http://localhost:3000
```

---

# Dashboard Panels Created

## Total Requests

```promql
sum(http_requests_total)
```

---

## Requests Per Second

```promql
sum(rate(http_requests_total[1m]))
```

---

## Service Status

```promql
up{job="payment-service"}
```

---

## Running Pods

```promql
count(kube_pod_status_ready{namespace="observability",condition="true"})
```

---

## CPU Usage

```promql
rate(container_cpu_usage_seconds_total{namespace="observability"}[1m])
```

---

# Phase 3: Centralized Logging

## Problem

Each pod stores logs independently.

Checking logs manually:

```bash
kubectl logs <pod-name>
```

becomes difficult as pods scale.

---

## Why Loki?

Loki stores logs efficiently and integrates with Grafana.

---

## Loki Installation

Added repo:

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

Installed:

```bash
helm install loki grafana/loki \
-n logging \
--create-namespace
```

Verify:

```bash
kubectl get pods -n logging
```

---

# Alloy

## Why Alloy?

Alloy collects logs from Kubernetes and forwards them to Loki.

Installed:

```bash
helm install alloy grafana/alloy \
-n logging
```

---

## Verification

Check logs:

```bash
kubectl logs -n logging daemonset/alloy
```

---

# Grafana + Loki Integration

Datasource:

```text
Loki
```

Query Example:

```logql
{job="loki.source.kubernetes.pods"}
```

Filtered Payment Service Logs:

```logql
{instance="observability/payment-service-xxxxx:payment-service"}
```

Observed:

```text
GET /metrics HTTP/1.1 200
```

This proved centralized logging was working.

---

# Phase 4: Metrics Server

## Why?

HPA requires CPU and memory metrics.

Install:

```bash
minikube addons enable metrics-server
```

Verify:

```bash
kubectl top nodes
kubectl top pods -n observability
```

---

# Phase 5: Horizontal Pod Autoscaler (HPA)

## Why?

Automatically scales pods based on resource usage.

Without HPA:

```text
Traffic ↑
Pods stay same
Application may become slow
```

With HPA:

```text
Traffic ↑
CPU ↑
Pods ↑
```

---

## Resource Requests and Limits

Added:

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

Reason:

HPA calculates utilization based on requested resources.

---

## Create HPA

```bash
kubectl autoscale deployment payment-service \
--cpu-percent=50 \
--min=2 \
--max=6 \
-n observability
```

Verify:

```bash
kubectl get hpa -n observability
```

Result:

```text
cpu: 1%/50%
```

This confirmed HPA was receiving metrics correctly.

---

# Commands Frequently Used

View Pods:

```bash
kubectl get pods -A
```

Watch Pods:

```bash
kubectl get pods -w
```

Describe Resource:

```bash
kubectl describe <resource>
```

View Logs:

```bash
kubectl logs <pod-name>
```

Port Forward:

```bash
kubectl port-forward svc/<service> local:target
```

Metrics:

```bash
kubectl top pods
kubectl top nodes
```

---

# Key Learnings

- Docker image creation
- Kubernetes Deployments
- Kubernetes Services
- ReplicaSets
- Prometheus Metrics
- ServiceMonitor CRDs
- Grafana Dashboards
- Loki Log Aggregation
- Alloy Log Collection
- Metrics Server
- Horizontal Pod Autoscaling
- Helm Package Management
- Kubernetes Troubleshooting

---

# Real-World Relevance

This project represents a simplified version of production observability systems used by:

- Netflix
- Uber
- Spotify
- Airbnb
- Amazon
- Google

The same concepts scale to microservice architectures with hundreds of services.

---

# Resume Description

Built a cloud-native observability platform on Kubernetes using Prometheus, Grafana, Loki, Alloy, and HPA. Implemented custom application metrics, centralized logging, dashboard visualization, and autoscaling for a containerized Flask application. Configured ServiceMonitor-based metric scraping and Kubernetes resource monitoring using Metrics Server.

---

# Project Status

✅ Application Deployment

✅ Docker

✅ Kubernetes

✅ Prometheus

✅ Grafana

✅ ServiceMonitor

✅ Custom Metrics

✅ Loki

✅ Alloy

✅ Centralized Logging

✅ Metrics Server

✅ Horizontal Pod Autoscaling

✅ Monitoring Dashboard

🎉 Project Completed Successfully
