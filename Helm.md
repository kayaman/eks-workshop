# Helm

## Install the Helm CLI

```
cd environment
curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get > get_helm.sh
chmod +x get_helm.sh
./get_helm.sh
cd -
```
**Note:** Do not run `helm init` :pray:

## Configure Helm access with RBAC

```
cat <<EoF > environment/rbac.yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
EoF
```

```
kubectl apply -f environment/rbac.yaml
```

```
helm init --service-account tiller
```

## Deploy NGINX with Helm

```
helm repo update

helm search

helm search nginx

helm repo add bitnami https://charts.bitnami.com/bitnami

helm search nginx

helm search bitnami/nginx
```

```
helm install --help

helm install bitnami/nginx

kubectl describe deployment zeroed-cardinal-nginx

kubectl get pods -l app=zeroed-cardinal-nginx

kubectl get service zeroed-cardinal-nginx -o wide
```

## Clean up

```
helm list
helm delete --purge zeroed-cardinal
```

```
kubectl get pods -l app=zeroed-cardinal-nginx
kubectl get service zeroed-cardinal-nginx -o wide
```

## Create a Chart

```
cd environment
helm create eksdemo
```

Delete boilerplate:
```
rm -rf environment/eksdemo/templates/
rm environment/eksdemo/Chart.yaml
rm environment/eksdemo/values.yaml
```

Create own files:
```
cat <<EoF > environment/eksdemo/Chart.yaml
apiVersion: v1
appVersion: "1.0"
description: A Helm chart for EKS Workshop Microservices application
name: eksdemo
version: 0.1.0
EoF
```

Check https://eksworkshop.com/helm_root/helm_micro/customize/ for substitutions needed on the templates below:
```
# Create subfolders for each template type
mkdir -p environment/eksdemo/templates/deployment
mkdir -p environment/eksdemo/templates/service

# Copy and rename frontend manifests
cp environment/ecsdemo-frontend/kubernetes/deployment.yaml environment/eksdemo/templates/deployment/frontend.yaml
cp environment/ecsdemo-frontend/kubernetes/service.yaml environment/eksdemo/templates/service/frontend.yaml

# Copy and rename crystal manifests
cp environment/ecsdemo-crystal/kubernetes/deployment.yaml environment/eksdemo/templates/deployment/crystal.yaml
cp environment/ecsdemo-crystal/kubernetes/service.yaml environment/eksdemo/templates/service/crystal.yaml

# Copy and rename nodejs manifests
cp environment/ecsdemo-nodejs/kubernetes/deployment.yaml environment/eksdemo/templates/deployment/nodejs.yaml
cp environment/ecsdemo-nodejs/kubernetes/service.yaml environment/eksdemo/templates/service/nodejs.yaml
```

```
cat <<EoF > environment/eksdemo/values.yaml
# Default values for eksdemo.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

# Release-wide Values
replicas: 3
version: 'latest'

# Service Specific Values
nodejs:
  image: brentley/ecsdemo-nodejs
crystal:
  image: brentley/ecsdemo-crystal
frontend:
  image: brentley/ecsdemo-frontend
EoF
```

### Use the dry-run flag to test our templates

```
helm install --debug --dry-run --name workshop environment/eksdemo
```

## Deploy the chart

```
helm install --name workshop environment/eksdemo
```

```
kubectl get svc ecsdemo-frontend -o jsonpath="{.status.loadBalancer.ingress[*].hostname}"; echo
```

### Optional

```
# break the image name and upgrade
helm upgrade workshop environment/eksdemo

# check 
helm status workshop

# rollback the failed upgrade
helm history workshop

# rollback to the 1st revision
helm rollback workshop 1

helm status workshop
```

## Clean up

```
helm del --purge workshop
```

