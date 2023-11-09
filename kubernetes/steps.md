```
CLUSTER_NAME=spittest2
mkdir -p $CLUSTER_NAME
NG_NAME=my-mng
NG_STACK_NAME=eksctl-$CLUSTER_NAME-nodegroup-$NG_NAME

# Create an EKS cluster without nodegroup
eksctl create cluster \
    --name $CLUSTER_NAME \
    --without-nodegroup \
    --region us-east-2 \
    --kubeconfig=$HOME/$CLUSTER_NAME/$CLUSTER_NAME-eks-cluster.kubeconfig \
    --zones=us-east-2a,us-east-2b

eksctl get cluster --name $CLUSTER_NAME --region us-east-2
KUBECONFIG=$HOME/$CLUSTER_NAME/$CLUSTER_NAME-eks-cluster.kubeconfig

# Create Node group
eksctl create nodegroup \
  --cluster $CLUSTER_NAME \
  --name $NG_NAME \
  --node-type t2.medium \
  --nodes 2 \
  --nodes-min 2 \
  --nodes-max 5 \
  --region us-east-2

eksctl get nodegroup --cluster $CLUSTER_NAME --region us-east-2

# Create IAM Policy
# Obtain role from cluster creation events

# Continuing after: https://gist.github.com/jdluther2020/93806237769b3e8726fa6f1ae754a235#file-create-nodegroup-sh

ROLE=$(aws cloudformation list-stack-resources  --stack-name $NG_STACK_NAME | jq -r '.[] | .[] | select(.LogicalResourceId == "NodeInstanceRole").PhysicalResourceId')

# Confirm role
echo $ROLE

leftoverPolicyArn=$(aws iam list-policies --query 'Policies[?PolicyName==`AmazonEKSClusterAutoscalerPolicySpit`].Arn' --output text)
aws iam delete-policy --policy-arn $leftoverPolicyArn

# Create the policy
POLICY_ARN=$(aws iam create-policy \
    --policy-name AmazonEKSClusterAutoscalerPolicySpit \
    --policy-document \
'{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "autoscaling:DescribeAutoScalingGroups",
                "autoscaling:DescribeAutoScalingInstances",
                "autoscaling:DescribeLaunchConfigurations",
                "autoscaling:DescribeTags",
                "autoscaling:SetDesiredCapacity",
                "autoscaling:TerminateInstanceInAutoScalingGroup",
                "ec2:DescribeLaunchTemplateVersions"
            ],
            "Resource": "*",
            "Effect": "Allow"
        }
    ]
}' | jq -r '.[].Arn')

# Confirm policy arn
echo $POLICY_ARN

# Attach policy to the role
aws iam attach-role-policy \
--role-name $ROLE \
--policy-arn $POLICY_ARN

# Confirm role policies
aws iam list-attached-role-policies --role-name $ROLE

# Deploy k8s auto-scaler
rm -rf ca.yaml
# Prepare the manifest file with our cluster name and options
curl https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml | sed  "s/<YOUR CLUSTER NAME>/$CLUSTER_NAME\n            - --balance-similar-node-groups\n            - --skip-nodes-with-system-pods=false\n            - --- --scale-down-unneeded-time=60s/g" > ca.yaml

KUBECONFIG=$HOME/$CLUSTER_NAME/$CLUSTER_NAME-eks-cluster.kubeconfig
# Create deployment
kubectl apply -f ./ca.yaml

# Confirm deployment is done
kubectl get deploy -n kube-system -o wide

# Confirm autoscaler pod is running
kubectl get pods -n kube-system -l app=cluster-autoscaler

# View Cluster Autoscaler logs to see it's monitoring cluster load.
kubectl -n kube-system logs -f deployment.apps/cluster-autoscaler

kubectl apply -f https://raw.githubusercontent.com/thecloudgarage/spit/main/kubernetes/metrics-server.yaml
kubectl apply -f https://raw.githubusercontent.com/thecloudgarage/spit/main/kubernetes/website.yaml
kubectl apply -f kubectl apply -f https://raw.githubusercontent.com/thecloudgarage/spit/main/kubernetes/website-hpa.yaml
kubectl get all -n demo

### TO DELETE....
eksctl delete nodegroup \
  --cluster $CLUSTER_NAME \
  --name $NG_NAME \
  --region us-east-2

aws cloudformation list-stacks --stack-status-filter DELETE_IN_PROGRESS --region us-east-2

eksctl delete cluster $CLUSTER_NAME --region us-east-2

aws cloudformation list-stacks --stack-status-filter DELETE_IN_PROGRESS --region us-east-2

#REF: https://awstip.com/this-code-works-kubernetes-cluster-autoscaler-on-amazon-eks-c2d059022e1c
```

