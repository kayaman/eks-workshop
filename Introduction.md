# Introduction

## Install Kubernetes tools

### Create the default ~/.kube directory for storing kubectl configuration

```
mv ~/.kube ~/._kube # I backed up my old one
mkdir -p ~/.kube
```

### Install kubectl (for MacOS)

```
brew install kubernetes-cli
```

### Install AWS IAM Authenticator

```
go get -u -v github.com/kubernetes-sigs/aws-iam-authenticator/cmd/aws-iam-authenticator
sudo mv ~/go/bin/aws-iam-authenticator /usr/local/bin/aws-iam-authenticator
```

### Install JQ and envsubst

```
brew install jq gettext
brew link --force gettext
echo 'export PATH="/usr/local/opt/gettext/bin:$PATH"' >> ~/.zshrc
```

### Verify the binaries are in the path and executable

```
for command in kubectl aws-iam-authenticator jq envsubst
  do
    which $command &>/dev/null && echo "$command in path" || echo "$command NOT FOUND"
  done
```

## Configure and verify account

```
export ACCOUNT_ID=$(aws sts get-caller-identity --output text --query Account)

echo "export ACCOUNT_ID=${ACCOUNT_ID}" >> ~/.zshrc
echo "export AWS_REGION=us-east-1" >> ~/.zshrc

aws configure set default.region us-east-1
aws configure get default.region
```

## Validate the IAM role

```
aws sts get-caller-identity
```

## SSH key

```
aws ec2 import-key-pair --key-name "eksworkshop" --public-key-material file://~/.ssh/id_rsa.pub
```

## `eksctl`

```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/download/latest_release/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp

sudo mv -v /tmp/eksctl /usr/local/bin

eksctl version
```

## Launch

```
eksctl create cluster --name=eksworkshop-eksctl --nodes=3 --node-ami=auto --region=us-east-1
```

## Test the cluster

```
kubectl get nodes

INSTANCE_PROFILE_NAME=$(aws iam list-instance-profiles | jq -r '.InstanceProfiles[].InstanceProfileName' | grep nodegroup)
INSTANCE_PROFILE_ARN=$(aws iam get-instance-profile --instance-profile-name $INSTANCE_PROFILE_NAME | jq -r '.InstanceProfile.Arn')
ROLE_NAME=$(aws iam get-instance-profile --instance-profile-name $INSTANCE_PROFILE_NAME | jq -r '.InstanceProfile.Roles[] | .RoleName')
echo "export ROLE_NAME=${ROLE_NAME}" >> ~/.zshrc
echo "export INSTANCE_PROFILE_ARN=${INSTANCE_PROFILE_ARN}" >> ~/.zshrc
```

## Deploy the official Kubernetes Dashboard

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
```

```
kubectl proxy --port=8080 --address='0.0.0.0' --disable-filter=true &
```

```
open http://localhost:8080//api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/
```

```
aws-iam-authenticator token -i eksworkshop-eksctl --token-only |pbcopy
```

## Deploy Node.js backend API

```
cd environment/ecsdemo-nodejs
kubectl apply -f kubernetes/deployment.yaml
kubectl apply -f kubernetes/service.yaml
cd -
```

```
kubectl get deployment ecsdemo-nodejs
```

## Deploy Crystal backend API

```
cd environment/ecsdemo-crystal
kubectl apply -f kubernetes/deployment.yaml
kubectl apply -f kubernetes/service.yaml
cd -
```

```
kubectl get deployment ecsdemo-crystal
```

## Check for the ELB service role existance

```
aws iam get-role --role-name "AWSServiceRoleForElasticLoadBalancing" || aws iam create-service-linked-role --aws-service-name "elasticloadbalancing.amazonaws.com"
```

## Deploy the frontend service

```
cd environment/ecsdemo-frontend
kubectl apply -f kubernetes/deployment.yaml
kubectl apply -f kubernetes/service.yaml
cd -
```

```
kubectl get deployment ecsdemo-frontend
```

## Find the service address

```
kubectl get service ecsdemo-frontend
```

```
kubectl get service ecsdemo-frontend -o wide
```

```
ELB=$(kubectl get service ecsdemo-frontend -o json | jq -r '.status.loadBalancer.ingress[].hostname')
curl -m3 -v $ELB
```

## Scale the backend services

```
kubectl get deployments
```

```
kubectl scale deployment ecsdemo-nodejs --replicas=3
kubectl scale deployment ecsdemo-crystal --replicas=3
```

```
kubectl get deployments
```

## Scale the frontend service

```
kubectl get deployments
kubectl scale deployment ecsdemo-frontend --replicas=3
kubectl get deployments
```

## Clean up the applications

```
cd environment/ecsdemo-frontend
kubectl delete -f kubernetes/service.yaml
kubectl delete -f kubernetes/deployment.yaml
cd -

cd environment/ecsdemo-crystal
kubectl delete -f kubernetes/service.yaml
kubectl delete -f kubernetes/deployment.yaml
cd -

cd environment/ecsdemo-nodejs
kubectl delete -f kubernetes/service.yaml
kubectl delete -f kubernetes/deployment.yaml
cd -
```
