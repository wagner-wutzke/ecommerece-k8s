# Ecommerce Microservices Deployment

In this project I created Kubernetes manifest files for deploying the ecommerce stack in two different ways:
1. Locally on a running existing cluster (e.g. minikube)
2. On Amazon EKS Cloud.

The manifests are organized in two mains folders, according to the target deployment environment.
- **eks**: for Amazon EKS
- **local**: for local cluster

Additionally, I added manifest files for installing a EFK stack (Elasticsearch FluentBit, 
Kibana) on the same cluster, so that logging and monitoring are centralized.
The manifests are to be found in the `efk` folder.

The steps to install the EFK stack are [here](eks/efk/README.md) to be found.