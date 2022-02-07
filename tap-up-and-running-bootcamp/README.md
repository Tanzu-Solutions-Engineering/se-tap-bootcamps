# TAP Up and Running bootcamp

This repository contains material for the SE TAP Up and Running bootcamp.

## Prerequisites for attending the bootcamp
- Be very comfortable in working with Kubernetes and knows how Kubernetes work (if that's not the case, it's recommended to do a CKAD certification first)
- Completed the **TAP Overview** workshop at: http://via.vmware.com/tap-demo
- Watched James Urquart's whiteboard pitch video (current version: https://via.vmw.com/EdB3)
- An account for https://network.tanzu.vmware.com
- Has access to a public could and has provisioned:
  - A Kubernetes cluster (GKE, AKS, EKS)
    - Commands to provision a GKE, AKS, or EKS cluster can be found here in the next section
    - (WIP) Terraform scripts to bootstrap the infra for TAP can be found here: https://github.com/Tanzu-Solutions-Engineering/se-tap-bootcamp-automation
  - A Docker V2 API compatibel container registry (GCR, ACR, Harbor, DockerHub, ..)
    - It’s not recommended to use AWS ECR as container registry due to its lack of support for long-lived credentials! 
- The privileges for a (sub-)domain to create wildcard DNS entries for e.g. *.cnrs.example.com, *.learningcenter.example.com, *.example.com
  - **Will be provided by the instructor, if the SE doesn't have a private domain!**
## Provision a Kubernetes cluster

The scripts are currently only validated with GKE and AWS EKS, and Azure AKS!

### GKE

There is an [issue](https://jira.eng.vmware.com/browse/TANZUSC-821) with GKE Zonal type clusters during the installation where the cluster is overloaded even if there are enough resources available and it goes several times into a repair state / requests to the Kubernetes API timeout because Zonal clusters have a single control plane in a single zone. You have to wait for some time until every sub-package is in `ReconcileSucceeded` state. 
For a better experience it's **recommended to switch to a Regional type cluster**.

With the following commands, you can provision a Regional type cluster with the [gcloud CLI](https://cloud.google.com/sdk/docs/install).
```
REGION=europe-west3
CLUSTER_ZONE=europe-west3-a
gcloud beta container clusters create tap --region $REGION --cluster-version "1.21.5-gke.1302" --machine-type "e2-standard-4" --num-nodes "4" --node-locations $CLUSTER_ZONE --enable-pod-security-policy
gcloud container clusters get-credentials tap --zone $CLUSTER_ZONE
```
Configure Pod Security Policies so that Tanzu Application Platform controller pods can run as root.
```
kubectl create clusterrolebinding tap-psp-rolebinding --group=system:authenticated --clusterrole=gce:podsecuritypolicy:privileged
```

### AWS EKS
With the following commands, you can provision a cluster with the [aws CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html).
```
aws configure
export CLUSTER_NAME=tap-demo

# Create EKS cluster role
export CLUSTER_SERVICE_ROLE=$(aws iam create-role --role-name "${CLUSTER_NAME}-eks-role" --assume-role-policy-document='{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"eks.amazonaws.com"},"Action": "sts:AssumeRole"}]}' --output text --query 'Role.Arn')
aws iam attach-role-policy --role-name "${CLUSTER_NAME}-eks-role" --policy-arn arn:aws:iam::aws:policy/AmazonEKSServicePolicy
aws iam attach-role-policy --role-name "${CLUSTER_NAME}-eks-role" --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy

# Create VPC
export STACK_ID=$(aws cloudformation create-stack --stack-name ${CLUSTER_NAME} --template-url https://amazon-eks.s3.us-west-2.amazonaws.com/cloudformation/2020-10-29/amazon-eks-vpc-private-subnets.yaml --output text --query 'StackId')
aws cloudformation wait stack-create-complete --stack-name ${CLUSTER_NAME}
export SECURITY_GROUP=$(aws cloudformation describe-stacks --stack-name ${CLUSTER_NAME} --query "Stacks[0].Outputs[?OutputKey=='SecurityGroups'].OutputValue" --output text)
export SUBNET_IDS=$(aws cloudformation describe-stacks --stack-name ${CLUSTER_NAME} --query "Stacks[0].Outputs[?OutputKey=='SubnetIds'].OutputValue" --output text)

# Create EKS cluster
aws eks create-cluster --name ${CLUSTER_NAME} --kubernetes-version 1.21 --role-arn "${CLUSTER_SERVICE_ROLE}" --resources-vpc-config subnetIds="${SUBNET_IDS}",securityGroupIds="${SECURITY_GROUP}"
aws eks wait cluster-active --name ${CLUSTER_NAME}

# Create EKS worker node role
export WORKER_SERVICE_ROLE=$(aws iam create-role --role-name "${CLUSTER_NAME}-eks-worker-role" --assume-role-policy-document='{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"ec2.amazonaws.com"},"Action": "sts:AssumeRole"}]}' --output text --query 'Role.Arn')
aws iam attach-role-policy --role-name "${CLUSTER_NAME}-eks-worker-role" --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
aws iam attach-role-policy --role-name "${CLUSTER_NAME}-eks-worker-role" --policy-arn arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
aws iam attach-role-policy --role-name "${CLUSTER_NAME}-eks-worker-role" --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly

# Create EKS worker nodes
aws eks create-nodegroup --cluster-name ${CLUSTER_NAME} --kubernetes-version 1.21 --nodegroup-name "${CLUSTER_NAME}-node-group" --disk-size 500 --scaling-config minSize=4,maxSize=4,desiredSize=4 --subnets $(echo $SUBNET_IDS | sed 's/,/ /g') --instance-types t3a.xlarge --node-role ${WORKER_SERVICE_ROLE}
aws eks wait nodegroup-active --cluster-name ${CLUSTER_NAME} --nodegroup-name ${CLUSTER_NAME}-node-group

aws eks update-kubeconfig --name ${CLUSTER_NAME}
```

### AKS Azure
With the following commands, you can provision a cluster with the [az CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli).

```
export CLUSTER_NAME=tap-demo
az login
az group create --location germanywestcentral --name ${CLUSTER_NAME}

# Add pod security policies support preview (required for learningcenter)
az extension add --name aks-preview
az feature register --name PodSecurityPolicyPreview --namespace Microsoft.ContainerService
# Wait until the status is "Registered"
az feature list -o table --query "[?contains(name, 'Microsoft.ContainerService/PodSecurityPolicyPreview')].{Name:name,State:properties.state}"
az provider register --namespace Microsoft.ContainerService

az aks create --resource-group ${CLUSTER_NAME} --name ${CLUSTER_NAME} --node-count 4 --enable-addons monitoring --node-vm-size Standard_DS3_v2 --node-osdisk-size 500 --enable-pod-security-policy

az aks get-credentials --resource-group ${CLUSTER_NAME} --name ${CLUSTER_NAME}

kubectl create clusterrolebinding tap-psp-rolebinding --group=system:authenticated --clusterrole=psp:privileged
```
