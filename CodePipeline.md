# CI/CD with Codepipeline

## Create IAM role

```
cd environment

TRUST="{ \"Version\": \"2012-10-17\", \"Statement\": [ { \"Effect\": \"Allow\", \"Principal\": { \"AWS\": \"arn:aws:iam::${ACCOUNT_ID}:root\" }, \"Action\": \"sts:AssumeRole\" } ] }"

echo '{ "Version": "2012-10-17", "Statement": [ { "Effect": "Allow", "Action": "eks:Describe*", "Resource": "*" } ] }' > /tmp/iam-role-policy

aws iam create-role --role-name EksWorkshopCodeBuildKubectlRole --assume-role-policy-document "$TRUST" --output text --query 'Role.Arn'

aws iam put-role-policy --role-name EksWorkshopCodeBuildKubectlRole --policy-name eks-describe --policy-document file:///tmp/iam-role-policy

cd -
```

## Modify AWS-Auth ConfigMap

```
ROLE="    - rolearn: arn:aws:iam::$ACCOUNT_ID:role/EksWorkshopCodeBuildKubectlRole\n      username: build\n      groups:\n        - system:masters"

kubectl get -n kube-system configmap/aws-auth -o yaml | awk "/mapRoles: \|/{print;print \"$ROLE\";next}1" > /tmp/aws-auth-patch.yml

kubectl patch configmap/aws-auth -n kube-system --patch "$(cat /tmp/aws-auth-patch.yml)"
```

### Fork a sample repository

https://github.com/rnzsgh/eks-workshop-sample-api-service-go


## Generate a GitHub access token

asdf

## CodePipeline setup


[CloudFormation template](https://console.aws.amazon.com/cloudformation/home?#/stacks/create/review?stackName=eksws-codepipeline&templateURL=https://s3.amazonaws.com/eksworkshop.com/templates/master/ci-cd-codepipeline.cfn.yml)

And check the pictures on the workshop site.

## Trigger new release

Change something on GitHub and :pray:

## Cleanup

```
kubectl delete deployments hello-k8s

kubectl delete services hello-k8s
```

Go to AWS Management Console and delete ECR repository and the CloudFormation stack.