# Creating an EKS Cluster in AWS

A cluster in EKS is composed by a **Control Plane** and Managed Nodes Group in AWS 
**Cloud Formation Stack**.
The Node Groups are responsible for running the application pods.

## Requirements
1. You need an AWS account. I used a Free Tier Account for testing. Amazon gives you 6 months and 
U$200 credits to use.
2. `Kubectl` and `eksctl` installed on your machine.
3. Downloaded project from GitHub.



## Create a Cluster using the CLI command

Go to your CLI and login:
```
aws login
```
You will be redirected to the AWS login page in your browser.\
Login with your credentials.


### Option 1: Using the CLI only
For a straightforward deployment with default settings, run the following command.\
This provisions an EKS cluster along with a managed node group of EC2 worker nodes. \
⚠️ Please be aware that when choosing this option, you will have to install the `AWS EBS CSI Driver` 
plugin at your own. This brought me a lot of headache during my experiments.

```bash
eksctl create cluster \
  --name ecommerce \
  --region us-east-2 \
  --nodegroup-name workers \
  --node-type m7i-flex.large \
  --version 1.37 \
  --nodes 4 \
  --nodes-min 3 \
  --nodes-max 5 \
  --node-volume-size=20 \
  --managed \
  --verbose 4
```

### Option 2: Create a Cluster Using a YAML Config File
__For production environments, maintaining your cluster layout as code is ideal.__\
Run the command providing the configuration file:
```bash
eksctl create cluster -f ./eks/cluster/cluster.yaml
```

This cluster manifest file also contains the installation of the `aws-ebs-csi-driver` and 
`eks-pod-identity-agent` plugins. I tried to install them separatedly and it was a very painful 
experience. So setting this option in the manifest file makes things much easier and 
straightforward. 

It is necessary to use the `m7i-flex.large` instance type if you need more than 2GB RAM available on
cluster node. This is the case here, since we are going to install Elasticsearch and have almost
40 pods running later.

## Verify the Cluster installation
The deployment process usually takes 15 to 25 minutes. \ 
Once `eksctl` finishes, it automatically updates your local kubeconfig file so your `kubectl` \
client can communicate with the new cluster.
Run these validation commands to check your resources:

```bash
# Verify connection and view node details
kubectl get all 

# Check system workloads running inside the cluster
kubectl get pods -A
```
<br/>

***
## Deploying the workload
The files being referenced here are available in my GitHub.

1. Deploy the `ecommerce` resources.
```bash
kubectl apply -f ./eks/ecommerce
```

2.  Check ecommerce resources: deployments, replicasets, pods, etc.
```bash
kubectl get all -n ecommerce 
```

```bash
# check ecommerce pod logs: 
kubectl logs <pod-name> -n ecommerce
```
You should see something like this output.
```bash
wagner@wagner-b:~/Documents/work/ecommerce-k8s/k8s$ kubectl get all -n ecommerce
NAME                                     READY   STATUS    RESTARTS   AGE
pod/customers-service-59d555b497-k2wbt   1/1     Running   0          2m23s
pod/inventory-service-f57956c54-wd4j5    1/1     Running   0          2m23s
pod/invoices-service-74d7866b4-jqkks     1/1     Running   0          2m21s
pod/kafka-broker-0                       1/1     Running   0          2m20s
pod/kafka-ui-554444ddb4-qc6hf            1/1     Running   0          2m24s
pod/orders-service-58f5c785bf-ltzdp      1/1     Running   0          2m24s
pod/payments-service-5587b759bc-wwlf7    1/1     Running   0          2m22s
pod/shipments-service-576bfb968d-hlcfq   1/1     Running   0          2m21s

NAME                        TYPE           CLUSTER-IP       EXTERNAL-IP                                                               PORT(S)                                         AGE
service/customers-service   ClusterIP      10.100.68.179    <none>                                                                    9010/TCP                                        2m28s
service/inventory-service   ClusterIP      10.100.250.110   <none>                                                                    9020/TCP                                        2m28s
service/invoices-service    ClusterIP      10.100.124.242   <none>                                                                    9040/TCP                                        2m26s
service/kafka-broker        NodePort       10.100.44.175    <none>                                                                    29092:32605/TCP,9093:30983/TCP,9092:30092/TCP   2m30s
service/kafka-ui            LoadBalancer   10.100.175.102   ac613502bf0f447b18ac827898a10beb-1343367526.us-east-2.elb.amazonaws.com   8080:31578/TCP                                  2m30s
service/orders-service      LoadBalancer   10.100.28.242    a8197449f6e634a6f8b5bd163885b31c-1203696367.us-east-2.elb.amazonaws.com   9000:31441/TCP                                  2m29s
service/payments-service    ClusterIP      10.100.102.101   <none>                                                                    9030/TCP                                        2m27s

NAME                                READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/customers-service   1/1     1            1           2m24s
deployment.apps/inventory-service   1/1     1            1           2m24s
deployment.apps/invoices-service    1/1     1            1           2m22s
deployment.apps/kafka-ui            1/1     1            1           2m25s
deployment.apps/orders-service      1/1     1            1           2m25s
deployment.apps/payments-service    1/1     1            1           2m23s
deployment.apps/shipments-service   1/1     1            1           2m22s

NAME                                           DESIRED   CURRENT   READY   AGE
replicaset.apps/customers-service-59d555b497   1         1         1       2m25s
replicaset.apps/inventory-service-f57956c54    1         1         1       2m25s
replicaset.apps/invoices-service-74d7866b4     1         1         1       2m23s
replicaset.apps/kafka-ui-554444ddb4            1         1         1       2m26s
replicaset.apps/orders-service-58f5c785bf      1         1         1       2m26s
replicaset.apps/payments-service-5587b759bc    1         1         1       2m24s
replicaset.apps/shipments-service-576bfb968d   1         1         1       2m23s

NAME                            READY   AGE
statefulset.apps/kafka-broker   1/1     2m22s
```

<br/>

***
## How to Clean Up
To prevent ongoing AWS charges, cleanly delete the cluster and all underlying CloudFormation-managed
stacks with a single command:
```bash
# If created via CLI parameters
eksctl delete cluster --name ecommerce --region us-east-2 --wait --disable-nodegroup-eviction

# If created via a YAML configuration file
eksctl delete cluster -f cluster.yaml --wait --disable-nodegroup-eviction
```
<br/>

***
## Links
- https://devopscube.com/create-aws-eks-cluster-eksctl/
- https://docs.kuberocketci.io/docs/operator-guide/infrastructure-providers/aws/ebs-csi-driver
- https://github.com/kubernetes-sigs/aws-ebs-csi-driver/blob/master/docs/install.md
- https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html#managing-ebs-csi
- https://devopscube.com/provsion-persistent-volume-on-eks/