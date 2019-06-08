# Monitoring using Prometheus and Grafana

(WIP: something went wrong)

## Deploy Prometheus

```
kubectl create namespace prometheus
```
```
helm install stable/prometheus \
        --name prometheus \
        --namespace prometheus \
        --set alertmanager.persistentVolume.storageClass="gp2" \
        --set server.persistentVolume.storageClass="gp2"
```

```
kubectl get all -n prometheus
```

```
kubectl port-forward -n prometheus deploy/prometheus-server 8080:9090
```

## Deploy Grafana

```
kubectl create namespace grafana
```
```
helm install stable/grafana \
    --name grafana \
    --namespace grafana \
    --set persistence.storageClassName="gp2" \
    --set datasources.apiVersion=1 \
    --set datasources.name=Prometheus \
    --set datasources.type=prometheus \
    --set datasources.url=http://prometheus-server.prometheus.svc.cluster.local \
    --set datasources.access=proxy \
    --set datasources.isDefault=true \
    --set service.type=LoadBalancer
```
```
helm install stable/grafana \
        --name grafana \
        --namespace grafana \
        --set persistence.storageClassName="gp2" \
        --set service.type=LoadBalancer
```
```
kubectl get all -n grafana
```

```
export ELB=$(kubectl get svc -n grafana grafana -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
```

```
open "http://${ELB}"
```

Get the 'admin'password:
```
kubectl get secret --namespace grafana grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```