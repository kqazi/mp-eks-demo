# Running Paid EKS Products

- Launched EKS support for pay-as-you-go (PAYG) container products on 9/25/19
- PAYG container products enable customers to pay only for what they use, similar to AWS service consumption
- IAM Roles for Service Accounts is a dependency to run PAYG container products on EKS

## Demo

### Prerequisites

1. Latest [AWS CLI](https://aws.amazon.com/cli/) installed.
2. Configure AWS CLI by running `aws configure`. The access and secret key must be mapped to an IAM user with a policy that allows cluster creation via `eksctl`. For demo purposes **only** use the Adminstrator role `arn:aws:iam::aws:policy/AdministratorAccess` to prevent permission issues. You would not use this level of permissions in a production enviornment. Also set the AWS region as `us-east-2`. You can double check this by running `aws configure get region`
3. [eksctl](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html) installed
4. [Helm and Tiller](https://docs.aws.amazon.com/eks/latest/userguide/helm.html) configured 


### Configure EKS Cluster

This will spin up a brand new EKS cluster based in the AWS region you configured the AWS CLI to use above. This process takes 10-20 mins to complete. Note: If you run this command more than once with the same EKS cluster name it will fail.

```
eksctl create cluster \
	--name mp-eks-irsa \
	--version 1.14 \
	--nodegroup-name standard-workers \
	--node-type t3.medium \
	--nodes 3 \
	--nodes-min 1 \
	--nodes-max 3 \
	--node-ami auto
```

### Configure IAM to K8 Service Account Mapping

1. Create IAM Policy that enables Paid Container product billing on EKS.

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "aws-marketplace:RegisterUsage"
            ],
            "Resource": "*",
            "Effect": "Allow"
        }
    ]
}
```

2. Enable IAM Roles for Service Accounts

This command will create a K8 service account, IAM Role based on policy in step #1 and annotate the K8 service account with the IAM Role. 

```
eksctl create iamserviceaccount \
  --name solodev-serviceaccount \
  --namespace default \
  --cluster mp-eks-irsa-2 \
  --attach-policy-arn [iam-policy-arn-from-step-1] \
  --approve
```

1. Subscribe to EKS enabled Paid Container product on AWS Marketplace [Solodev DCX Enterprise Edition for Kubernetes] (https://aws.amazon.com/marketplace/pp/B07XV951M6?qid=1571433963481&sr=0-2&ref_=srh_res_product_title)
2. Configure IAM Roles for Service Accounts on EKS cluster 
Launch product using Helm
