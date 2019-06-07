# Implement logging with EFK

Together, Fluentd, Elasticsearch and Kibana is also known as “EFK stack”. Fluentd will forward logs from the individual instances in the cluster to a centralized logging backend (CloudWatch Logs) where they are combined for higher-level reporting using ElasticSearch and Kibana.

- [Elasticsearch](https://www.elastic.co/products/elasticsearch) is a distributed, RESTful search and analytics engine.
- [Fluentd](https://www.fluentd.org/) is an open source data collector providing a unified logging layer, supported by 500+ plugins connecting to many types of systems.
- [Kibana](https://www.elastic.co/products/kibana) lets you visualize your Elasticsearch data.

## Configure IAM policy for worker nodes

```
test -n "$ROLE_NAME" && echo ROLE_NAME is "$ROLE_NAME" || echo ROLE_NAME is not set
```

```
mkdir environment/iam_policy

cat <<EoF > environment/iam_policy/k8s-logs-policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "logs:DescribeLogGroups",
                "logs:DescribeLogStreams",
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": "*",
            "Effect": "Allow"
        }
    ]
}
EoF

aws iam put-role-policy --role-name $ROLE_NAME --policy-name Logs-Policy-For-Worker --policy-document file://environment/iam_policy/k8s-logs-policy.json
```

Check:
```
aws iam get-role-policy --role-name $ROLE_NAME --policy-name Logs-Policy-For-Worker
```

## Provision an Elasticsearch cluster

**Note:** this cluster has an open access policy which will need to be locked down in production environments.

```
aws es create-elasticsearch-domain \
        --domain-name kubernetes-logs \
        --elasticsearch-version 6.3 \
        --elasticsearch-cluster-config \
        InstanceType=m4.large.elasticsearch,InstanceCount=2 \
        --ebs-options EBSEnabled=true,VolumeType=standard,VolumeSize=100 \
        --access-policies '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"AWS":["*"]},"Action":["es:*"],"Resource":"*"}]}'
```

```
aws es describe-elasticsearch-domain --domain-name kubernetes-logs --query 'DomainStatus.Processing'
```

## Deploy Fluentd

```
mkdir environment/fluentd
cd environment/fluentd
wget https://eksworkshop.com/logging/deploy.files/fluentd.yml
cd -
```

```
kubectl apply -f environment/fluentd/fluentd.yml
```

```
kubectl get pods -w --namespace=kube-system
```

## Configure Cloudwatch logs and Kibana

```
cat <<EoF > environment/iam_policy/lambda.json
{
   "Version": "2012-10-17",
   "Statement": [
   {
     "Effect": "Allow",
     "Principal": {
        "Service": "lambda.amazonaws.com"
     },
   "Action": "sts:AssumeRole"
   }
 ]
}
EoF
```
```
aws iam create-role --role-name lambda_basic_execution --assume-role-policy-document file://environment/iam_policy/lambda.json
```
```
aws iam attach-role-policy --role-name lambda_basic_execution --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

## Cleanup logging

```
kubectl delete -f environment/fluentd/fluentd.yml
rm -rf environment/fluentd/
aws es delete-elasticsearch-domain --domain-name kubernetes-logs
aws logs delete-log-group --log-group-name /eks/eksworkshop-eksctl/containers
```