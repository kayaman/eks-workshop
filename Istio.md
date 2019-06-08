# Istio

Istio is a completely open source service mesh that layers transparently onto existing distributed applications. It’s also a platform, including APIs, that let it integrate into any logging platform, or telemetry or policy system.

Let’s review in more detail what each of the components that make up this service mesh are.

- Envoy

Processes the inbound/outbound traffic from inter-service and service-to-external-service transparently.

- Pilot

Pilot provides service discovery for the Envoy sidecars, traffic management capabilities for intelligent routing (e.g., A/B tests, canary deployments, etc.), and resiliency (timeouts, retries, circuit breakers, etc.)

- Mixer

Mixer enforces access control and usage policies across the service mesh, and collects telemetry data from the Envoy proxy and other services.

- Citadel

Citadel provides strong service-to-service and end-user authentication with built-in identity and credential management.


## Install Istio

```
cd environment

curl -L https://git.io/getLatestIstio | sh -

cd istio-1.0.8

mv -v bin/istioctl /usr/local/bin/
```

### Define service account for Tiller

```
kubectl apply -f install/kubernetes/helm/helm-service-account.yaml
```

### Install Istio CRDs

```
helm install install/kubernetes/helm/istio-init --name istio-init --namespace istio-system
```
```
kubectl get crds --namespace istio-system | grep 'istio.io'
```

### Install Istio

```
helm install install/kubernetes/helm/istio --name istio --namespace istio-system --set global.configValidation=false --set sidecarInjectorWebhook.enabled=false --set grafana.enabled=true --set servicegraph.enabled=true
```
```
kubectl get svc -n istio-system
```
```
kubectl get pods -n istio-system
```

## Deploy sample apps

```
kubectl apply -f <(istioctl kube-inject -f samples/bookinfo/platform/kube/bookinfo.yaml)
```
```
kubectl get pod,svc
```
```
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
```
kubectl get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' -n istio-system ; echo
```

## Intelligent routing

```
kubectl apply -f samples/bookinfo/networking/destination-rule-all.yaml

kubectl get destinationrules -o yaml
```

```
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
```

```
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml

kubectl get virtualservices reviews -o yaml
```

```
spec:
  hosts:
  - reviews
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v1
```

```
kubectl apply -f samples/bookinfo/networking/virtual-service-ratings-test-delay.yaml

kubectl get virtualservice ratings -o yaml
```

```
spec:
  hosts:
  - ratings
  http:
  - fault:
      delay:
        fixedDelay: 7s
        percent: 100
    match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: ratings
        subset: v1
  - route:
    - destination:
        host: ratings
        subset: v1
```

```
kubectl apply -f samples/bookinfo/networking/virtual-service-ratings-test-abort.yaml

kubectl get virtualservice ratings -o yaml
```

```
spec:
  hosts:
  - ratings
  http:
  - fault:
      abort:
        httpStatus: 500
        percent: 100
    match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: ratings
        subset: v1
  - route:
    - destination:
        host: ratings
        subset: v1
```

```
kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml

kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-50-v3.yaml

kubectl get virtualservice reviews -o yaml
```

```
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 50
    - destination:
        host: reviews
        subset: v3
      weight: 50
```

## Monitor & Visualize

### Collecting new telemetry data

```
curl -LO https://eksworkshop.com/servicemesh/deploy.files/istio-telemetry.yaml

kubectl apply -f istio-telemetry.yaml
```

```
kubectl -n istio-system get svc prometheus

kubectl -n istio-system get svc grafana
```

```
kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=grafana -o jsonpath='{.items[0].metadata.name}') 8080:3000 &
```

```
export SMHOST=$(kubectl get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].hostname} ' -n istio-system)

SMHOST="$(echo -e "${SMHOST}" | tr -d '[:space:]')"

while true; do curl -o /dev/null -s "${SMHOST}/productpage"; done
```

## Cleanup

```
kubectl delete -f istio-telemetry.yaml
```

```
kubectl delete -f samples/bookinfo/networking/virtual-service-all-v1.yaml

kubectl delete -f samples/bookinfo/networking/destination-rule-all.yaml
```

```
kubectl delete -f samples/bookinfo/networking/bookinfo-gateway.yaml

kubectl delete -f samples/bookinfo/platform/kube/bookinfo.yaml
```

```
helm delete --purge istio
helm delete --purge istio-init
```
