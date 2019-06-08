# Autoscaling

**HPA**: Horizontal Pod Autoscaler
**CA**: Cluster Autoscaler

## Horizontal Pod Autoscaler (HPA)

```
helm install stable/metrics-server \
     --name metrics-server \
     --version 2.0.4 \
     --namespace metrics
```

```
kubectl get apiservice v1beta1.metrics.k8s.io -o yaml
```

Response should look like this:
```
status:
  conditions:
  - lastTransitionTime: 2018-10-15T15:13:13Z
    message: all checks passed
    reason: Passed
    status: "True"
    type: Available
``` 

### Deploy a sample app

```
kubectl run php-apache --image=k8s.gcr.io/hpa-example --requests=cpu=200m --expose --port=80
```

### Create an HPA resource

```
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
```

```
kubectl get hpa
```

### Generate load to trigger scaling

```
kubectl run -i --tty load-generator --image=busybox /bin/sh

while true; do wget -q -O - http://php-apache; done

kubectl get hpa -w
```


## Cluster Autoscaler (CA)

### Deploy a sample app

```
cat <<EoF> environment/cluster-autoscaler/nginx.yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-to-scaleout
spec:
  replicas: 1
  template:
    metadata:
      labels:
        service: nginx
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx-to-scaleout
        resources:
          limits:
            cpu: 500m
            memory: 512Mi
          requests:
            cpu: 500m
            memory: 512Mi
EoF
```

```
kubectl apply -f ~/environment/cluster-autoscaler/nginx.yaml
kubectl get deployment/nginx-to-scaleout
```

### Scale the ReplicaSet

```
kubectl scale --replicas=10 deployment/nginx-to-scaleout
```

Watch:
```
kubectl logs -f deployment/cluster-autoscaler -n kube-system
```

### Configure the Cluster Autoscaler (CA)

```
mkdir environment/cluster-autoscaler
cd environment/cluster-autoscaler
wget https://eksworkshop.com/scaling/deploy_ca.files/cluster_autoscaler.yml
```

### Configure the ASG

Get the ASG name from the Aws Management Console. Change `Min` and `Max` values.

### Configure the Cluster Autoscaler

Edit:
```
command:
  - ./cluster-autoscaler
  - --v=4
  - --stderrthreshold=info
  - --cloud-provider=aws
  - --skip-nodes-with-local-storage=false
  - --nodes=2:8:<AUTOSCALING GROUP NAME>
env:
  - name: AWS_REGION
    value: us-east-1
```
Note: 2:8 -> must match your planned Min:Max capacity

### Create an IAM Policy

```
test -n "$ROLE_NAME" && echo ROLE_NAME is "$ROLE_NAME" || echo ROLE_NAME is not set
```

```
mkdir environment/asg_policy
cat <<EoF > environment/asg_policy/k8s-asg-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "autoscaling:DescribeAutoScalingGroups",
        "autoscaling:DescribeAutoScalingInstances",
        "autoscaling:SetDesiredCapacity",
        "autoscaling:TerminateInstanceInAutoScalingGroup"
      ],
      "Resource": "*"
    }
  ]
}
EoF
aws iam put-role-policy --role-name $ROLE_NAME --policy-name ASG-Policy-For-Worker --policy-document file://environment/asg_policy/k8s-asg-policy.json
```

```
aws iam get-role-policy --role-name $ROLE_NAME --policy-name ASG-Policy-For-Worker
```

### Deploy the Cluster Autoscaler

```
kubectl apply -f environment/cluster-autoscaler/cluster_autoscaler.yml
```

Watch:
```
kubectl logs -f deployment/cluster-autoscaler -n kube-system
```

### Deploy a Sample App

```
cat <<EoF> environment/cluster-autoscaler/nginx.yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-to-scaleout
spec:
  replicas: 1
  template:
    metadata:
      labels:
        service: nginx
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx-to-scaleout
        resources:
          limits:
            cpu: 500m
            memory: 512Mi
          requests:
            cpu: 500m
            memory: 512Mi
EoF
```

```
kubectl apply -f environment/cluster-autoscaler/nginx.yaml
```

```
kubectl get deployment/nginx-to-scaleout
```

### Scale our ReplicaSet

```
kubectl scale --replicas=10 deployment/nginx-to-scaleout
```

Watch:
```
kubectl get pods -o wide --watch
```
(sample output:)
```
NAME                                 READY     STATUS    RESTARTS   AGE

nginx-to-scaleout-7cb554c7d5-2d4gp   0/1       Pending   0          11s
nginx-to-scaleout-7cb554c7d5-2nh69   0/1       Pending   0          12s
nginx-to-scaleout-7cb554c7d5-45mqz   0/1       Pending   0          12s
nginx-to-scaleout-7cb554c7d5-4qvzl   0/1       Pending   0          12s
nginx-to-scaleout-7cb554c7d5-5jddd   1/1       Running   0          34s
nginx-to-scaleout-7cb554c7d5-5sx4h   0/1       Pending   0          12s
nginx-to-scaleout-7cb554c7d5-5xbjp   0/1       Pending   0          11s
nginx-to-scaleout-7cb554c7d5-6l84p   0/1       Pending   0          11s
nginx-to-scaleout-7cb554c7d5-7vp7l   0/1       Pending   0          12s
nginx-to-scaleout-7cb554c7d5-86pr6   0/1       Pending   0          12s
nginx-to-scaleout-7cb554c7d5-88ttw   0/1       Pending   0          12s

```

Watch:
```
kubectl logs -f deployment/cluster-autoscaler -n kube-system
```

Also, check the EC2 fleet on AWS Management Console to see the ASG acting up.

## Cleaning up

```
kubectl delete -f environment/cluster-autoscaler/cluster_autoscaler.yml
kubectl delete -f environment/cluster-autoscaler/nginx.yaml
kubectl delete hpa,svc php-apache
kubectl delete deployment php-apache load-generator
```