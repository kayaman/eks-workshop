# AWS App Mesh

## Create the k8s app

### Clone the repo

```
cd environment
git clone https://github.com/aws/aws-app-mesh-examples
cd aws-app-mesh-examples/examples/apps/djapp/
```

### Setup permissions for the worker nodes

```
cat <<EoF > k8s-appmesh-worker-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "appmesh:DescribeMesh",
        "appmesh:DescribeVirtualNode",
        "appmesh:DescribeVirtualService",
        "appmesh:DescribeVirtualRouter",
        "appmesh:DescribeRoute",
        "appmesh:CreateMesh",
        "appmesh:CreateVirtualNode",
        "appmesh:CreateVirtualService",
        "appmesh:CreateVirtualRouter",
        "appmesh:CreateRoute",
        "appmesh:UpdateMesh",
        "appmesh:UpdateVirtualNode",
        "appmesh:UpdateVirtualService",
        "appmesh:UpdateVirtualRouter",
        "appmesh:UpdateRoute",
        "appmesh:ListMeshes",
        "appmesh:ListVirtualNodes",
        "appmesh:ListVirtualServices",
        "appmesh:ListVirtualRouters",
        "appmesh:ListRoutes",
        "appmesh:DeleteMesh",
        "appmesh:DeleteVirtualNode",
        "appmesh:DeleteVirtualService",
        "appmesh:DeleteVirtualRouter",
        "appmesh:DeleteRoute"
  ],
      "Resource": "*"
    }
  ]
}
EoF
```

```
aws iam put-role-policy --role-name $ROLE_NAME --policy-name AM-Policy-For-Worker --policy-document file://k8s-appmesh-worker-policy.json

aws iam get-role-policy --role-name $ROLE_NAME --policy-name AM-Policy-For-Worker
```

### Test permissions

```
sed -i'.old' -e 's/\"us-west-2\"/\"'$AWS_REGION'\"/' awscli.yaml
```
```
kubectl apply -f awscli.yaml
```

```
kubectl get jobs
```

```
kubectl logs jobs/awscli
```

The output should be something like:
```
{
    "meshes": []
}
```

### Create DJ app

```
kubectl apply -f 1_create_the_initial_architecture/1_prod_ns.yaml
kubectl apply -nprod -f 1_create_the_initial_architecture/1_initial_architecture_deployment.yaml
kubectl apply -nprod -f 1_create_the_initial_architecture/1_initial_architecture_services.yaml
```

Check:
```
kubectl get all -nprod
```

### Test DJ app

```
kubectl get pods -nprod -l app=dj
```

```
kubectl exec -nprod -it <your-dj-pod-name> bash
```

```
curl jazz-v1.prod.svc.cluster.local:9080;echo

curl metal-v1.prod.svc.cluster.local:9080;echo
```

## Create the App Mesh components

### About sidecars

They can be deployed in three different ways:

1. Before the "deployment", modifying the deployment's specs to include the sidecar. When deployed, the sidecar will run.
2. After the deployment, patching the container specs to include the sidecar specs. Old pods will be torn down, and the new ones would come up with the sidecar.
3. Through an Injector Controller, this way all subsequent deployments will come up with the sidecar automatically. This in not only quicker in the long run, but reduces chances of errors (typos, etc) and allow standardization.




## Porting DJ app to App Mesh


## Cleanuo