# Deploying the ecommerce backend to a cluster

A cluster in EKS is composed by a **Control Plane** and Managed Nodes Group in AWS **Cloud Formation Stack**.
The Node Groups are responsible for running the application pods.

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