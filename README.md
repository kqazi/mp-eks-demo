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

### Configure [IAM Roles for Service Accounts](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)

1. Create IAM Policy that enables Paid Container product billing on EKS. Copy the ARN for this policy as it will be used in step #2.

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

### Associate EKS OIDC Provider with IAM

This command will map the EKS OIDC provider that was created during cluster creation in IAM. You can verify that this is setup correctly by going to your EKS cluster in the AWS Console and ensuring that the "OpenID Connect provider URL" is configured in the IAM Console under "Identity Provders" section. 

```
eksctl utils associate-iam-oidc-provider \
  --name mp-eks-irsa \
  --approve
```

### MAP IAM Role to K8 Serivce Account 
This command will create a K8 service account, IAM Role based on policy in step #1 and annotate the K8 service account with the IAM Role. 

```
eksctl create iamserviceaccount \
  --name solodev-serviceaccount \
  --namespace default \
  --cluster mp-eks-irsa \
  --attach-policy-arn [iam-policy-arn-from-step-1] \
  --approve
```

Verify K8 Service account creation: `kubectl get sa solodev-serviceaccount -o yaml`. You should see output similar to the below. You'll notice that the IAM Role has been automatically created for you (using the policy supplied to the command above) and the K8 service account annotation with the IAM Role. **If the IAM Role is not in the annotation IAM Roles for Service Accounts will not work.**

```
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::[aws-account-id]:role/eksctl-mp-eks-irsa-2-addon-iamserviceaccount-Role1-1E5XV9GAJMF5A
  creationTimestamp: "2019-10-18T22:28:51Z"
  name: solodev-serviceaccount
  namespace: default
  resourceVersion: "7001"
  selfLink: /api/v1/namespaces/default/serviceaccounts/solodev-serviceaccount
  uid: a8f52cc4-f1f6-11e9-a317-06d79cb9eeea
secrets:
- name: solodev-serviceaccount-token-6k66t
```

### Verify IAM Role creation

Go to the IAM Console and verify that the role referenced in the service account annotation above exists, and that the policy attached to the role allows calls to `RegisterUsage`. You should this policy attached to the role:

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

### Subscribe and Launch EKS Product

Subscribe to EKS enabled Paid Container product on AWS Marketplace [Solodev DCX Enterprise Edition for Kubernetes] (https://aws.amazon.com/marketplace/pp/B07XV951M6?qid=1571433963481&sr=0-2&ref_=srh_res_product_title).This is a paid AWS Marketplace product that will charge based on a pay-as-you-go hourly rate.

After you subscribe run:

```
helm repo add charts 'https://raw.githubusercontent.com/techcto/charts/master/'
helm repo update
helm repo list
```

Next start up Tiller in the background in your local environment:

``` 
tiller -listen=localhost:44134 -storage=secret -logtostderr 
```

Now install SoloDev:

```
helm install --name solodev-dcx charts/solodev-dcx-aws \
  --set serviceAccountName='solodev-serviceaccount' \
  --set solodev.storage.className=gp2 \
  --set solodev.settings.appSecret=secret \
  --set solodev.settings.appPassword=password \
  --set solodev.settings.dbPassword=password
```

**NOTE:** If you get an error that Tiller can't be found make sure you setup Tiller correctly by following the steps here: https://docs.aws.amazon.com/eks/latest/userguide/helm.html

You can check the status of the deployment by runninug: `kubectl get pods`. You should see output similar to:

```
NAME                                   READY   STATUS    RESTARTS   AGE
solodev-dcx-mongo-6894cfcf99-b44mt     1/1     Running   0          2m31s
solodev-dcx-mysql-0                    2/2     Running   0          2m31s
solodev-dcx-redis-0                    1/1     Running   0          2m30s
solodev-dcx-solodev-779b466dc9-brqh9   1/1     Running   1          2m31s
solodev-dcx-ui-7dd69c7fb5-wrmgn        1/1     Running   0          2m31s
```
Once all pods are in a RUNNING state execute:

```
kubectl get svc solodev-dcx-ui -o yaml | grep hostname
```

You should see output similar to:
```
    - hostname: ac5ad876cfbe711e9a9ce02dcc828796-ec827f76ebc5baa2.elb.us-east-2.amazonaws.com
```

Copy this into your browser and you should see the SoloDev login page.
