# Kubernetes Observability Platform - Troubleshooting Journey

## Purpose

This document records every major issue encountered during the project, including the error observed, the root cause, the solution applied, and the lesson learned.

The goal is to understand not only how the project works but also how production issues were investigated and resolved.

---

# Issue 1: Prometheus Target Showing DOWN

## Symptom

Prometheus Targets page showed:

```text
0/2 UP
```

Error:

```text
received unsupported Content-Type "text/html; charset=utf-8"
```

---

## Root Cause

The Flask application's `/metrics` endpoint was returning normal HTML instead of Prometheus-formatted metrics.

Prometheus expects:

```text
text/plain; version=1.0.0
```

but received:

```text
text/html
```

---

## Investigation

Checked endpoint:

```bash
curl -I http://localhost:8082/metrics
```

Observed:

```text
Content-Type: text/html; charset=utf-8
```

---

## Fix

Modified metrics endpoint:

```python
from flask import Response
from prometheus_client import generate_latest, CONTENT_TYPE_LATEST

@app.route("/metrics")
def metrics():
    return Response(
        generate_latest(),
        mimetype=CONTENT_TYPE_LATEST
    )
```

Rebuilt image and restarted deployment.

---

## Lesson Learned

Prometheus does not scrape arbitrary HTTP responses.

Metrics endpoint must return Prometheus exposition format with correct Content-Type.

---

# Issue 2: Application Changes Not Reflecting

## Symptom

Updated source code but Prometheus still showed old behavior.

---

## Root Cause

Kubernetes was running an old container image.

Updating local source code does not automatically update running pods.

---

## Investigation

Checked running container:

```bash
kubectl exec -it <pod> -- cat /app/app.py
```

Found old code still running.

---

## Fix

Rebuild image:

```bash
docker build -t payment-service:v2 .
```

Load image:

```bash
minikube image load payment-service:v2
```

Restart deployment:

```bash
kubectl rollout restart deployment/payment-service -n observability
```

---

## Lesson Learned

Always verify what code is actually running inside the container.

---

# Issue 3: ServiceMonitor Not Scraping Metrics

## Symptom

Prometheus target remained DOWN.

---

## Root Cause

Metrics endpoint was misconfigured.

Prometheus could reach the application but could not parse the response.

---

## Investigation

Checked:

```bash
curl http://localhost:8082/metrics
```

Verified output format.

---

## Fix

Corrected metrics endpoint implementation.

---

## Lesson Learned

Network connectivity and scrape format are separate problems.

---

# Issue 4: Loki Datasource Failed to Connect

## Symptom

Grafana displayed:

```text
Unable to connect with Loki
```

---

## Root Cause

Version mismatch.

Grafana expected a newer Loki API while an older Loki version was running.

---

## Investigation

Checked image:

```bash
kubectl describe pod loki-0 -n logging
```

Observed:

```text
grafana/loki:2.6.1
```

---

## Fix

Removed old Loki deployment.

Installed newer Loki version.

Verified:

```bash
grafana/loki:3.6.7
```

---

## Lesson Learned

Always verify component versions when integrations fail.

---

# Issue 5: Loki Port Forward Returned 404

## Symptom

Opening:

```text
http://localhost:3100
```

returned:

```text
404 Not Found
```

---

## Root Cause

Loki is an API backend.

It does not provide a web UI.

---

## Fix

Configured Loki as Grafana datasource.

---

## Lesson Learned

Not every Kubernetes service provides a browser-accessible interface.

---

# Issue 6: Logs Appeared Only For Loki Canary

## Symptom

Grafana showed logs only from:

```text
loki-canary
```

No payment-service logs appeared.

---

## Root Cause

Alloy was installed but not configured to push logs to Loki.

---

## Investigation

Checked Alloy ConfigMap:

```bash
kubectl get configmap alloy -n logging -o yaml
```

Observed only discovery configuration.

No Loki write configuration existed.

---

## Fix

Added:

```hcl
loki.source.kubernetes
```

and

```hcl
loki.write
```

configuration.

Applied updated config.

Restarted Alloy.

---

## Lesson Learned

Installing an agent is not enough.

Collection pipeline must be explicitly configured.

---

# Issue 7: Alloy Logs Showed No Errors But No Logs Arrived

## Symptom

Alloy pods healthy.

No payment-service logs in Grafana.

---

## Root Cause

Discovery was working.

Forwarding pipeline was missing.

---

## Fix

Configured:

```hcl
forward_to = [loki.write.default.receiver]
```

---

## Lesson Learned

Healthy agents can still be doing nothing useful.

Always validate data flow end-to-end.

---

# Issue 8: HPA Showing CPU Unknown

## Symptom

```text
cpu: <unknown>/50%
```

---

## Root Cause

Deployment lacked CPU requests.

HPA calculates utilization using CPU requests.

Without requests, utilization cannot be computed.

---

## Investigation

Checked:

```bash
kubectl describe hpa payment-service -n observability
```

Observed:

```text
missing request for cpu
```

---

## Fix

Added:

```yaml
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
```

Restarted deployment.

---

## Lesson Learned

HPA requires resource requests.

---

# Issue 9: HPA Still Referencing Old Pods

## Symptom

HPA errors referenced deleted pod names.

---

## Root Cause

Old ReplicaSets still existed during rollout.

HPA temporarily evaluated metrics from terminating pods.

---

## Fix

Waited for rollout completion.

Verified new pods:

```bash
kubectl get pods -n observability
```

---

## Lesson Learned

During rolling updates, Kubernetes may temporarily reference old pods.

---

# Issue 10: Metrics Server Dependency

## Symptom

HPA could not retrieve resource metrics.

---

## Root Cause

Metrics Server was not installed.

---

## Fix

Enabled:

```bash
minikube addons enable metrics-server
```

Verified:

```bash
kubectl top nodes
kubectl top pods
```

---

## Lesson Learned

Prometheus metrics and Kubernetes resource metrics are different systems.

HPA depends on Metrics Server, not Prometheus.

---

# Biggest Takeaways

1. Always verify running container code.
2. Read error messages carefully.
3. Check Content-Type when debugging Prometheus.
4. Verify versions when tools fail to integrate.
5. Validate every step of a log pipeline.
6. HPA requires CPU requests.
7. Metrics Server is mandatory for CPU-based autoscaling.
8. Kubernetes troubleshooting is mostly systematic verification.
9. Never assume configuration is applied correctly.
10. Use kubectl describe extensively during debugging.

---

# Final Result

Successfully built and troubleshooted:

* Kubernetes Deployment
* Service
* Prometheus Monitoring
* Grafana Dashboards
* ServiceMonitor
* Loki Logging
* Alloy Log Collection
* Metrics Server
* Horizontal Pod Autoscaler

This troubleshooting journey provided practical experience with real-world Kubernetes observability issues and debugging workflows.
