# Health checks

Reference: [Liveness and Readiness probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/)

## Configure Liveness probe

```
mkdir -p environment/healthchecks
```

```
cat <<EoF > environment/healthchecks/liveness-app.yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-app
spec:
  containers:
  - name: liveness
    image: brentley/ecsdemo-nodejs
    livenessProbe:
      httpGet:
        path: /health
        port: 3000
      initialDelaySeconds: 5
      periodSeconds: 5
EoF
```
```
kubectl apply -f environment/healthchecks/liveness-app.yaml
```
```
kubectl get pod liveness-app
```
```
kubectl describe pod liveness-app
```

## Introduce a failure

```
kubectl exec -it liveness-app -- /bin/kill -s SIGUSR1 1
```

Watch:
```
kubectl describe pod liveness-app
```

```
kubectl get pod liveness-app
```

### How to check the status of the container health checks

```
kubectl logs liveness-app

# or

kubectl logs liveness-app --previous
```

## Configure Readiness probe

```
kubectl apply -f environment/healthchecks/readiness-deployment.yaml
```
```
kubectl get pods -l app=readiness-deployment
```
```
kubectl describe deployment readiness-deployment | grep Replicas:
```
```
kubectl exec -it <YOUR-READINESS-POD-NAME> -- rm /tmp/healthy
```
```
kubectl get pods -l app=readiness-deployment
```
``` 
kubectl describe deployment readiness-deployment | grep Replicas:
```

## Cleaning up

```
kubectl delete -f environment/healthchecks/liveness-app.yaml
kubectl delete -f environment/healthchecks/readiness-deployment.yaml
```