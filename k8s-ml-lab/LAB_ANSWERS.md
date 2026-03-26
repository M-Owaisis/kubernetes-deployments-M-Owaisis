# Lab Exercise Answers

## 1) The Environment Trap
No, the `/predict` output does not change immediately when `MODEL_NAME` is changed in the `ConfigMap`.

Reason: in this app, `MODEL_NAME` is loaded once at process start (`os.getenv(...)`) and stored in a Python variable. Updating the `ConfigMap` object in Kubernetes does not automatically restart existing pods or re-read environment variables inside the running Python process.

To see the new value, restart pods (for example `kubectl rollout restart deployment ml-deployment`) so containers start with the updated environment.

## 2) Security Audit
Command to view `API_KEY`:

```bash
kubectl get secret ml-auth -o jsonpath="{.data.API_KEY}" | base64 --decode
```

It is base64-encoded by default, not strongly encrypted by itself in Kubernetes Secret YAML output. Real at-rest encryption requires enabling Kubernetes secret encryption at the cluster level (encryption providers/KMS).

## 3) Scaling Logic
From the observed output, HPA scaled from 1 replica to 2 replicas after CPU crossed target:
- `cpu: 37%/30%`, replicas moved from `1` to `2`
- then CPU reached `184%/30%` and replicas scaled to `4` and then `5`

Observed second-pod trigger time in watch output was about 1 second between two consecutive HPA lines (`9m1s` to `9m2s`) after metrics were available and over target.

Why not instant in general: HPA works on periodic metrics collection and control loops (not per-request), and Kubernetes uses stabilization behavior (especially for scale-down) to avoid replica flapping.

## 4) Fault Tolerance / Desired State
After deleting a running pod (`kubectl delete pod <pod-name>`), Kubernetes immediately creates a replacement pod because the Deployment's desired state still requires the configured replica count.

This demonstrates the "Desired State" model: controllers continuously reconcile actual cluster state back to the declared state in the Deployment spec.

## 5) ML Implementation (`/health` + `livenessProbe`)
Completed.

- Added route in `app.py`:
  - `GET /health` returns HTTP 200 with `{"status":"ok"}`.
- Added `livenessProbe` in `manifest.yaml`:
  - HTTP GET probe to `/health` on port `5000`
  - `initialDelaySeconds: 10`
  - `periodSeconds: 10`

Apply updated manifest:

```bash
kubectl apply -f manifest.yaml
kubectl rollout restart deployment ml-deployment
kubectl get pods -w
```
