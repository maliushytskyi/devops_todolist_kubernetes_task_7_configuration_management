# Deploying ToDo app with ConfigMap and Secret

All resources belong to the `todoapp` namespace and are stored in the `.infrastructure/` directory.

## Prerequisites

- A running Kubernetes cluster (e.g. minikube) and `kubectl` configured for it
- The application image built from this repository (`settings.py` reads `SECRET_KEY` from the environment) and pushed to Docker Hub

```bash
kubectl cluster-info
kubectl get ns todoapp || kubectl create namespace todoapp
```

## Apply the changes

The ConfigMap and the Secret must be created before the Deployment, because the Deployment references them.

```bash
kubectl apply -f .infrastructure/configMap.yml
kubectl apply -f .infrastructure/secret.yml
kubectl apply -f .infrastructure/deployment.yml
```

Wait until the rollout is finished:

```bash
kubectl rollout status deployment/todoapp -n todoapp
```

## Validate the changes

### 1. Resources exist and the pod is running

```bash
kubectl get configmap todoapp-config -n todoapp
kubectl get secret todoapp-secret -n todoapp
kubectl get deployment todoapp -n todoapp
kubectl get pods -n todoapp
```

Expected: the ConfigMap and the Secret are listed, the Deployment is `READY 1/1`, the pod status is `Running`.

### 2. ConfigMap contains `PYTHONUNBUFFERED`

```bash
kubectl get configmap todoapp-config -n todoapp -o yaml
```

Expected: `data` contains the `PYTHONUNBUFFERED` key.

### 3. Secret contains `SECRET_KEY`

```bash
kubectl get secret todoapp-secret -n todoapp -o jsonpath='{.data.SECRET_KEY}' | base64 --decode
```

Expected: the secret value is printed (it is stored base64-encoded).

### 4. Environment variables are set in the container

```bash
kubectl describe deployment todoapp -n todoapp
kubectl exec -n todoapp deploy/todoapp -- printenv PYTHONUNBUFFERED
kubectl exec -n todoapp deploy/todoapp -- printenv SECRET_KEY
```

Expected: the `Environment` section lists both variables (from the ConfigMap and from the Secret), and `printenv` prints their values.

### 5. Application uses the Secret, not a hardcoded key

```bash
grep -n "SECRET_KEY" todolist/settings.py
```

Expected: the value is read from the environment (`os.environ["SECRET_KEY"]`), no literal key in the file.

### 6. Application works

```bash
kubectl logs -n todoapp deploy/todoapp --tail=20
kubectl port-forward -n todoapp deploy/todoapp 8000:8000
```

In another terminal:

```bash
curl -i http://localhost:8000/api/health
```

Expected: HTTP `200`, and the logs show output without delay (`PYTHONUNBUFFERED` is set).

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| `CreateContainerConfigError` | ConfigMap or Secret is missing, or its name or key in the Deployment does not match |
| `CrashLoopBackOff` | `SECRET_KEY` is missing in the container; see `kubectl logs` and `kubectl describe pod` |
| Changes to ConfigMap or Secret have no effect | `kubectl rollout restart deployment/todoapp -n todoapp` |

## Cleanup

```bash
kubectl delete -f .infrastructure/deployment.yml
kubectl delete -f .infrastructure/secret.yml
kubectl delete -f .infrastructure/configMap.yml
```