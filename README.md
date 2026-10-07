# Creating an EKS Cluster in AWS

A cluster in EKS is composed by a **Control Plane** and Managed Nodes Group in AWS **Cloud Formation Stack**.
The Node Groups are responsible for running the application pods.

## Creating a dedicated IAM user
In order to create and configure AWS resources over the CLI, it is recommended to create a 
dedicated IAM user. This can be done in the AWS IAM admin page.

I created a new user called `eks-ecommerce-user`.
It must have following Policies:
- AmazonEC2FullAccess
- AmazonEKSClusterPolicy
- AmazonVPCFullAccess
- AWSCloudFormationFullAccess
- IAMFullAccess
- AmazonEBSCSIDriverPolicyV2

After adding the policies, it is required to generate one access key to execute the login with
that user over the AWS CLI.

Configure the aws profile for the new user:
```
aws configure --profile eks-ecommerce-user

  AWS Access Key ID: AKIAWN...
  AWS Secret Access Key: VzkkoYJwV+EVC+5a42lV4Y4ERhAubr0oJCmCeWPl
  AWS region: us-east-2
  output format: json

export AWS_PROFILE=eks-ecommerce-user  # export the user profile
aws sts get-caller-identity  # check if the profile has been set

```

## Option 1: Create a Cluster using the CLI command
For a straightforward deployment with default settings, run the following command. 
This provisions an EKS cluster along with a managed node group of EC2 worker nodes:

```bash
eksctl create cluster \
  --name ecommerce \
  --region us-east-2 \
  --nodegroup-name workers \
  --node-type t3.small \
  --version 1.37 \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3 \
  --node-volume-size=20 \
  --managed \
  --verbose 4
```

## Option 2: Create a Cluster Using a YAML Config File
For production environments, maintaining your cluster layout as code is ideal.
1. Create a file named `cluster.yaml`:

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: ecommerce
  region: us-east-2
  version: "1.37"

managedNodeGroups:
  - name: workers
    instanceType: t3.small
    desiredCapacity: 2
    minSize: 1
    maxSize: 3
    volumeSize: 20
    ssh:
      allow: true
      publicKeyPath: ~/.ssh/aws_key.pub
```

2. Then run the command providing the configuration file:
```bash
eksctl create cluster -f ./k8s/aws/cluster/cluster.yaml --wait
```

## Verify the Cluster Connection
The deployment process usually takes 15 to 25 minutes. Once eksctl finishes, it automatically 
updates your local kubeconfig file so your kubectl client can communicate with the new cluster.
Run these validation commands to check your resources:
```bash
# Describe the cluster configs
aws eks describe-cluster --name ecommerce

# Verify connection and view node details
kubectl get nodes -o wide

# Check system workloads running inside the cluster
kubectl get pods -A
```

## Adding EKS ESB CSI Driver Addon
In order to use ESB volumes with Kubernetes, it is necessary to install this addon.

1. Find the proper role name for attach to the driver policy:
```
EXACT_ROLE_NAME=$(aws iam list-roles \
  --query "Roles[?starts_with(RoleName, 'eksctl-ecommerce-nodegroup-man-NodeInstanceRole-')].RoleName" \
  --output text)
echo "Found Role: $EXACT_ROLE_NAME" 
```

2. Attach the role policy for the EBS CSI Driver.
```bash
aws iam attach-role-policy \
  --role-name $EXACT_ROLE_NAME \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --region us-east-2
```

3. Add the Addon from the Cluster management page.
4. Alternatively you can add it via CLI:
```
eksctl create addon --name aws-ebs-csi-driver --cluster ecommerce

eksctl get addons --cluster ecommerce
```

__Note__: This grants all nodes in the group broad EBS permissions. It works but is not
least-privilege. Migrate to IRSA or EKS Pod Identity as soon as possible.

## How to Clean Up
To prevent ongoing AWS charges, cleanly delete the cluster and all underlying CloudFormation-managed 
stacks with a single command:
```bash
# If created via CLI parameters
eksctl delete cluster --name ecommerce --region us-east-2 --wait --disable-nodegroup-eviction

# If created via a YAML configuration file
eksctl delete cluster -f cluster.yaml --wait --disable-nodegroup-eviction
```

## Deploying the workload
```bash
kubectl apply -f ./k8s/aws/

kubectl get pods -n ecommerce -o wide

kubectl get svc -n ecommerce -o wide

kubectl logs <pod-name> -n ecommerce
```

## Links
- https://devopscube.com/create-aws-eks-cluster-eksctl/