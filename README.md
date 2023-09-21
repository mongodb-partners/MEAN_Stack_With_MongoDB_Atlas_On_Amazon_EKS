# Simplifying Application Modernization with Amazon EKS & MongoDB Atlas for developers
Deploy a MEAN Stack application with MongoDB Atlas as the backend on the Amazon EKS cluster

## Introduction
This is a technical repo to demonstrate the application deployment using MongoDB Atlas and AWS EKS. This tutorial is intended for those who want to

1. Serverless Application Deployment for Production Environment
2. Production deployment to auto-scale, HA, and Security
3. Agile development of application modernization
4. Deployment of containerized application in AWS
5. Want to try out the AWS EKS and MongoDB Atlas

## MongoDB Atlas
MongoDB Atlas is an all-purpose database having features like Document Model, Geo-spatial, Time-series, hybrid deployment, and multi-cloud services. It evolved as a "Developer Data Platform", intended to reduce the developer's workload on development and management of the database environment. It also provides a free tier to test out the application/database features.

## Amazon Elastic Kubernetes Services (EKS)
Amazon Elastic Kubernetes Service (Amazon EKS) is a managed Kubernetes service to run Kubernetes in the AWS cloud and on-premises data centers. In the cloud, Amazon EKS automatically manages the availability and scalability of the Kubernetes control plane nodes responsible for scheduling containers, managing application availability, storing cluster data, and other key tasks. With Amazon EKS, you can take advantage of all the performance, scale, reliability, and availability of AWS infrastructure, as well as integrations with AWS networking and security services. In this lab, you'll learn how to deploy a sample dockerized MEAN (MongoDB, Express, Angular, Node) stack application on a scalable and secure AWS infrastructure using Amazon EKS.


## Architecture Diagram

<img width="837" alt="image" src="https://github.com/mongodb-partners/MEAN_Stack_With_MongoDB_Atlas_On_Amazon_EKS/assets/101570105/94c73699-849d-4767-a050-3da6c9430c64">

## Step by Step EKS Deployment

### **Step1: Set up the MongoDB Atlas cluster**
MongoDB Atlas provides a free cluster setup. Pls follow the link to set up the free cluster

### **Step2: Create a new IAM user**
Login from Isengard. Create a new username and password. User Name: eks-user This user must have AdministratorAccess permission.

Go to IAM and select Users on the left panel, then select the user name you just created. Select Security credentials. At the Access keys section, click the Create access key button.

Download the CSV file to save the access key and secret access key.

### Step3: Use CloudShell to deploy K8 VPC eks-bastion EC2 instance
Log in to the AWS console with the username and password created in the previous step. This lab is designed to work in us-east-1 region. Make sure you move to us-east-1 region for all operations.

Open the AWS Console and click the CloudShell icon. CloudShell provides an environment that allows you to run AWS CLI commands.


<img width="837" alt="image" src="https://github.com/mongodb-partners/MEAN_Stack_With_MongoDB_Atlas_On_Amazon_EKS/assets/101570105/ef2c8a4c-81bd-4131-9c2a-9ac8ca602af4">


Create a directory for ec2-user

    cd /home
    sudo mkdir ec2-user
    sudo chown cloudshell-user ec2-user
    sudo chgrp cloudshell-user ec2-user
    cd ec2-user
    mkdir environment
    cd environment


Install Terraform to deploy the VPC network architecture

    sudo yum install -y yum-utils
    sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/AmazonLinux/hashicorp.repo
    sudo yum install terraform -y


Verify terraform is installed


    terraform version

Terraform v1.5.6
on linux_amd64

We will use Terraform to create the VPC, subnet, and NAT gateway.

Checkout the terraform networking module from GitHub

    cd /home/ec2-user/environment
    
    git clone https://github.com/haibzhou/terraform

Cloning into 'terraform'...

remote: Enumerating objects: 27, done.

remote: Counting objects: 100% (27/27), done.

remote: Compressing objects: 100% (15/15), done.

remote: Total 27 (delta 9), reused 27 (delta 9), pack-reused 0

Receiving objects: 100% (27/27), 15.46 KiB | 2.58 MiB/s, done.

Resolving deltas: 100% (9/9), done.


Create the necessary parameters to terraform.tfvars that we need to create the VPC networking architecture that will be used for EKS.

    cd terraform
    cat > terraform.tfvars <<EOF
    //AWS 
    region      = "us-east-1"
    environment = "k8s"
    
    /* module networking */
    vpc_cidr             = "10.0.0.0/16"
    public_subnets_cidr  = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"] //List of Public subnet cidr range
    private_subnets_cidr = ["10.0.11.0/24", "10.0.12.0/24", "10.0.13.0/24"] //List of private subnet cidr range
    EOF

Run the following commands to create VPC, subnet, IGW, and NAT gateway.

Initialize the terraform

    terraform init

Setup the plan 

    terraform plan

Apply the plan

    terraform apply

Check VPC named k8-vpc is created

<img width="832" alt="image" src="https://github.com/mongodb-partners/MEAN_Stack_With_MongoDB_Atlas_On_Amazon_EKS/assets/101570105/9cffbf6f-0200-4748-b0ca-41b1ec43b201">

Deploy EC2 instance eks-bastion in k8s-vpc. We will use this EC2 instance for the rest of this lab.


    export VPC_ID=$(aws ec2 describe-vpcs --filters "Name=tag:Name,Values=k8s-vpc" |jq -r .Vpcs[].VpcId )

    aws ec2 create-security-group \
        --group-name eke-bastion-sg \
        --description "AWS ec2 CLI Demo SG" \
        --tag-specifications 'ResourceType=security-group,Tags=[{Key=Name,Value=eks-bastion-sg}]' \
        --vpc-id ${VPC_ID}


    export SG_ID=$(aws ec2 describe-security-groups     --filters Name=tag:Name,Values=eks-bastion-sg     --query "SecurityGroups[*].{ID:GroupId}" |jq -r .[0].ID)

    aws ec2 authorize-security-group-ingress --group-id ${SG_ID} --protocol tcp --port 22 --cidr 0.0.0.0/0


    export PUBLIC_SUBNETS=$(aws ec2 describe-subnets --filters "Name=tag:Name,Values=k8s-us-east-1a-public-subnet" | jq -r .Subnets[].SubnetId)


    aws ec2 run-instances \
        --image-id ami-04cb4ca688797756f \
        --count 1 \
        --instance-type t2.micro \
        --security-group-ids ${SG_ID} \
        --subnet-id ${PUBLIC_SUBNETS} \
        --associate-public-ip-address \
        --block-device-mappings "[{\"DeviceName\":\"/dev/sdf\",\"Ebs\":{\"VolumeSize\":30,\"DeleteOnTermination\":false}}]" \
        --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=eks-bastion}]' 'ResourceType=volume,Tags=[{Key=Name,Value=eks-baston-disk}]'


Go to the AWS console to verify the EC2 instance is created and wait for the instance state to become Running.

The rest of the workshop will work from the eks-bastion ec2 instance.

### Step4: Create EKS Cluster
Open the AWS console and connect to this eks-bastion EC2 instance. Configure the AWS Access Key and Security Key on eks-bastion EC2 instance. EKS deployment will take about 20 minutes. The ssh session will be expired if no keys are entered. We need to configure the keep alive packet to keep the session alive.

    cat >~/.ssh/config <<EOF
     Host *
      ServerAliveInterval 50
      ServerAliveCountMax 3
    EOF

Configure AWS credential for AWS CLI

    aws configure
    AWS Access Key ID [****************K262]: 
    AWS Secret Access Key [****************C48w]: 
    Default region name [us-west-2]: us-east-1
    Default output format [None]:

Install eksctl for creating and managing your EKS cluster

    curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
    sudo mv /tmp/eksctl /usr/local/bin

Verify eksctl is installed.

    eksctl version


Prepare the cluster configuration file First we need to grab the private subnet id of our terraform-generated VPC resources so that we can deploy our EKS cluster to the VPC topology we just created. We should be able to achieve this by using jq command to parse the terraform output command.


    # export VPC
    export VPC_ID=$(aws ec2 describe-vpcs --filters "Name=tag:Name,Values=k8s-vpc" |jq -r .Vpcs[].VpcId)

    # export public subnets
    export PRIVATE_SUBNETS_ID_A=$(aws ec2 describe-subnets --filters "Name=tag:Name,Values=k8s-us-east-1a-private-subnet" | jq -r .Subnets[].SubnetId)
    export PRIVATE_SUBNETS_ID_B=$(aws ec2 describe-subnets --filters "Name=tag:Name,Values=k8s-us-east-1b-private-subnet" | jq -r .Subnets[].SubnetId)
    export PRIVATE_SUBNETS_ID_C=$(aws ec2 describe-subnets --filters "Name=tag:Name,Values=k8s-us-east-1c-private-subnet" | jq -r .Subnets[].SubnetId)

Create an EKS cluster configuration file We shall create an EKS configuration file called 'eks-cluster.yaml` to pass the corresponding parameters to provision the cluster in the VPC we just created.

    mkdir /home/ec2-user/environment
    
    cat > /home/ec2-user/environment/eks-cluster.yaml <<EOF
    ---
    apiVersion: eksctl.io/v1alpha5
    kind: ClusterConfig
    metadata:
      name: web-host-on-eks
      region: us-east-1 # Specify the aws region
      version: "1.27"
    privateCluster: # Allows configuring a fully-private cluster in which no node has outbound internet access, and private access to AWS services is enabled via VPC endpoints
      enabled: false
    vpc:
      id: "${VPC_ID}" # Specify the VPC_ID to the eksctl command
      subnets: # Creating the EKS master nodes to a completely private environment
        private:
          private-us-east-1a: 
            id: "${PRIVATE_SUBNETS_ID_A}"
          private-us-east-1b:
            id: "${PRIVATE_SUBNETS_ID_B}"
          private-us-east-1c:
            id: "${PRIVATE_SUBNETS_ID_C}"
    managedNodeGroups: # Create a managed node group in private subnets
    - name: managed
      labels:
        role: worker
      instanceType: t3.small
      minSize: 3
      desiredCapacity: 3
      maxSize: 10
      privateNetworking: true
      volumeSize: 50
      volumeType: gp2
      volumeEncrypted: true
      iam:
        withAddonPolicies: 
          autoScaler: true # enables IAM policy for cluster-autoscaler
          albIngress: true 
          cloudWatch: true 
      # securityGroups:
      #    attachIDs: ["sg-1", "sg-2"]
      ssh:
          allow: true
          publicKeyPath: ~/.ssh/id_rsa.pub
          # new feature for restricting SSH access to certain AWS security group IDs
      subnets:
        - private-us-east-1a
        - private-us-east-1b
        - private-us-east-1c
    cloudWatch:
      clusterLogging:
        # enable specific types of cluster control plane logs
        enableTypes: ["all"]
        # all supported types: "api", "audit", "authenticator", "controllerManager", "scheduler"
        # supported special values: "*" and "all"
    addons: # explore more on doc about EKS addons: https://docs.aws.amazon.com/eks/latest/userguide/eks-add-ons.html
    - name: vpc-cni # no version is specified so it deploys the default version
      attachPolicyARNs:
        - arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
    - name: coredns
      version: latest # auto discovers the latest available
    - name: kube-proxy
      version: latest
    iam:
      withOIDC: true # Enable OIDC identity provider for plugins, explore more on doc: https://docs.aws.amazon.com/eks/latest/userguide/authenticate-oidc-identity-provider.html
      serviceAccounts: # create k8s service accounts(https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/) and associate with IAM policy, see more on: https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html
      - metadata:
          name: aws-load-balancer-controller # create the needed service account for aws-load-balancer-controller while provisioning ALB/ELB by k8s ingress api
          namespace: kube-system
        wellKnownPolicies:
          awsLoadBalancerController: true
      - metadata:
          name: cluster-autoscaler # create the CA needed service account and its IAM policy
          namespace: kube-system
          labels: {aws-usage: "cluster-ops"}
        wellKnownPolicies:
          autoScaler: true
    EOF

Before running the eksctl command, we need to generate the default ssh key in your EC2 instance.

    ssh-keygen

Please press enter to keep all the input values as default. Next, we shall run the eksctl command to create our cluster by using the yaml configuration file we just created.

    eksctl create cluster -f /home/ec2-user/environment/eks-cluster.yaml

During the waiting time, you can navigate to the AWS Console, and check the CloudFormation service page, you should be able to check the status of the CloudFormation stack in progress. The stack name should be called eksctl-web-host-on-eks-cluster. This will take about 20 minutes.

<img width="832" alt="image" src="https://github.com/mongodb-partners/MEAN_Stack_With_MongoDB_Atlas_On_Amazon_EKS/assets/101570105/51101ec2-ddda-4083-ba60-a25370cf3188">


Access the EKS Console After successfully creating the EKS cluster, you should be able to navigate to the EKS cluster detail page to explore different tabs for your cluster.

<img width="832" alt="image" src="https://github.com/mongodb-partners/MEAN_Stack_With_MongoDB_Atlas_On_Amazon_EKS/assets/101570105/b9ff04a3-199b-4ba8-8817-a1febba1e170">

Access your EKS cluster

**Install kubectl to manage your cluster.**

    sudo bash -c "cat <<EOF > /etc/yum.repos.d/kubernetes.repo
    [kubernetes]
    name=Kubernetes
    baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
    enabled=1
    gpgcheck=1
    repo_gpgcheck=1
    gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
    EOF"

    sudo yum install -y kubectl
    kubectl version 

Client Version: v1.28.1
Kustomize Version: v5.0.4-0.20230601165947-6ce0bf390ce3

Set EKS cluster security group inbound rules

We need to allow inbound TCP traffic from our eks-bastion EC2 instance to the EKS cluster. To achieve this, we can navigate to the EKS cluster console and switch to the Networking tab, and open the Additional security groups to add the inbound rules of the security group. image

<img width="832" alt="image" src="https://github.com/mongodb-partners/MEAN_Stack_With_MongoDB_Atlas_On_Amazon_EKS/assets/101570105/1f1f75b0-84e6-4da0-b5a5-339a725ee049">


Edit inbound rule to allow traffic originating from k8-bastion security group name aws-EC2 instance SG named eks-bastion-sg. image

<img width="832" alt="image" src="https://github.com/mongodb-partners/MEAN_Stack_With_MongoDB_Atlas_On_Amazon_EKS/assets/101570105/7a07c48f-ac53-4e3a-8a90-ea7ca4130000">


Test that your eks-bastion EC2 instance can access EKS cluster control plane.

    kubectl get nodes

NAME                          STATUS   ROLES    AGE   VERSION
ip-10-0-11-186.ec2.internal   Ready    <none>   38m   v1.27.4-eks-8ccc7ba
ip-10-0-12-195.ec2.internal   Ready    <none>   38m   v1.27.4-eks-8ccc7ba
ip-10-0-13-247.ec2.internal   Ready    <none>   38m   v1.27.4-eks-8ccc7ba


    kubectl get pod -A


NAMESPACE     NAME                       READY   STATUS    RESTARTS   AGE
kube-system   aws-node-fz27q             1/1     Running   0          39m
kube-system   aws-node-htjdn             1/1     Running   0          39m
kube-system   aws-node-n5bm6             1/1     Running   0          39m
kube-system   coredns-64d7849ffb-gt9bm   1/1     Running   0          37m
kube-system   coredns-64d7849ffb-qbcgl   1/1     Running   0          37m
kube-system   kube-proxy-bnktr           1/1     Running   0          37m
kube-system   kube-proxy-mnhxh           1/1     Running   0          37m
kube-system   kube-proxy-qbxkp           1/1     Running   0          37m


**Install toolkit – Docker**

    sudo dnf update
    sudo dnf install docker -y
    sudo usermod -a -G docker ec2-user
    newgrp docker
    sudo systemctl start docker
    sudo systemctl enable docker
    docker version


Client:
 Version:           20.10.23
 API version:       1.41
 Go version:        go1.18.9
 Git commit:        7155243
 Built:             Tue Apr 11 22:56:36 2023
 OS/Arch:           linux/amd64
 Context:           default
 Experimental:      true

**Test docker engine**

    docker run hello-world

Hello from Docker!


**Install toolkit – Helm**


    curl -sSL https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash

    helm version


version.BuildInfo{Version:"v3.12.3", GitCommit:"3a31588ad33fe3b89af5a2a54ee1d25bfe6eaa5e", GitTreeState:"clean", GoVersion:"go1.20.7"}

**Install NodeJS Runtime**

    curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.34.0/install.sh | bash
    . ~/.nvm/nvm.sh
    nvm install --lts

**Install git on EC2 instance**

    sudo yum install git -y

**Get the application code**

    cd /home/ec2-user/environment
    
    git clone https://github.com/mongodb-partners/MEANStack_with_Atlas_on_Fargate.git
    
    cd ~/environment/MEANStack_with_Atlas_on_Fargate/code/MEANSTACK/partner-meanstack-atlas-fargate

Copy Mongodb database connection for node.js Login to mongod.com and go to Database. Select database Demo, click Connect  Drivers


<img width="832" alt="image" src="https://github.com/mongodb-partners/MEAN_Stack_With_MongoDB_Atlas_On_Amazon_EKS/assets/101570105/a78bed0b-1ad8-43e9-a1fe-0a14a86fbc80">

<img width="803" alt="image" src="https://github.com/mongodb-partners/MEAN_Stack_With_MongoDB_Atlas_On_Amazon_EKS/assets/101570105/cb04a23f-6c03-4e23-8846-9b8499a2c631">


Copy the connection string and replace the password with the correct password.

mongodb+srv://eks-user:<password>@demo.xxxxxx.mongodb.net/?retryWrites=true&w=majority

    cd ~/environment/MEANStack_with_Atlas_on_Fargate/code/MEANSTACK/partner-meanstack-atlas-fargate/server/

Set ~/environment/MEANStack_with_Atlas_on_Fargate/code/MEANSTACK/partner-meanstack-atlas-fargate/server/.env to the connection string and verify

    cat ~/environment/MEANStack_with_Atlas_on_Fargate/code/MEANSTACK/partner-meanstack-atlas-fargate/server/.env

ATLAS_URI=mongodb+srv://eks-user:<password>@demo.xxxxxx.mongodb.net/?retryWrites=true&w=majoritywq

**Install Docker compose on eks-baston EC2 instance**

    sudo curl -L https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose

    sudo chmod +x /usr/local/bin/docker-compose

    docker-compose version

Docker Compose version v2.21.0

**Create a repository on Elastic Container Registry(ECR)**

    aws ecr create-repository \
      --repository-name partner-meanstack-atlas-eks-client \
      --image-scanning-configuration scanOnPush=true \
      --region us-east-1
    
    aws ecr create-repository \
      --repository-name partner-meanstack-atlas-eks-server \
      --image-scanning-configuration scanOnPush=true \
      --region us-east-1

Open the code and update docker-compose.yaml To deploy our frontend and backend applications, we will make use of docker-compose to build and push images to ECR.

Get VPC ID and ACCOUNT ID

    export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
    export VPC_ID=$(aws ec2 describe-vpcs --filters "Name=tag:Name,Values=k8s-vpc" |jq -r .Vpcs[].VpcId )
    cd /home/ec2-user/environment/MEANStack_with_Atlas_on_Fargate/code/MEANSTACK/partner-meanstack-atlas-fargate

    cat > docker-compose.yml <<EOF
      
    version: "3"
    
    x-aws-vpc: ${VPC_ID}
    
    
    services:
      client:
        image: ${ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com/partner-meanstack-atlas-eks-client
        platform: linux/amd64
        build: ./client
        ports:
          - 8080
      server:
        build: ./server
        image: ${ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com/partner-meanstack-atlas-eks-server
        platform: linux/amd64
        ports:
          - 5200
    EOF

Build the docker image and push it to ECR

    aws ecr get-login-password --region us-east-1| docker login --username AWS --password-stdin ${ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com
    docker context use default
    docker-compose build
    docker-compose push

Prepare for the EKS namespace manifest

    cat > /home/ec2-user/environment/deploy_namespace.yaml <<EOF
    apiVersion: v1
    kind: Namespace # create the namespace for this application
    metadata:
      name: mongodb
    EOF

Create EKS namespace mongodb

    cd ~/environment
    kubectl apply -f deploy_namespace.yaml

Verify namespace is created

    kubectl get namespaces

NAME              STATUS   AGE
default           Active   7h7m
kube-node-lease   Active   7h7m
kube-public       Active   7h7m
kube-system       Active   7h7m
mongodb           Active   5h42m

Prepare manifest for the server application.

      cat > /home/ec2-user/environment/deploy_server.yaml <<EOF
      apiVersion: apps/v1
      kind: Deployment 
      metadata:
        name: server-deployment
        namespace: mongodb
      spec:
        selector:
          matchLabels:
            app: server
        template:
          metadata:
            labels:
              app: server
          spec:
            containers:
            - name: server
              image: ${ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com/partner-meanstack-atlas-eks-server:latest # specify your ECR repository
              ports:
              - containerPort: 5200
              resources:
                  limits:
                    cpu: 500m
                  requests:
                    cpu: 250m
      ---
      apiVersion: v1
      kind: Service
      metadata:
        name: server-service
        namespace: mongodb
        labels:
          app: server
      spec:
        selector:
          app: server
        ports:
          - protocol: TCP
            port: 5200 
            targetPort: 5200
      EOF

Deploy server application

    kubectl apply -f deploy_server.yaml 

deployment.apps/server-deployment created
service/server-service created


Verify server application is created. Make sure the status is Running.

    kubectl get pods -n mongodb

NAME                                                    READY   STATUS    RESTARTS   AGE
server-deployment-55ccd58d44-hpdnm   1/1        Running               0          33s

Because we shall deploy the Ingress(ALB) to the public subnets to serve the private application as the single traffic door, we need to grab the public subnets to the template.

    # export public subnets
    export PUBLIC_SUBNETS_ID_A=$(aws ec2 describe-subnets --filters "Name=tag:Name,Values=k8s-us-east-1a-public-subnet" | jq -r .Subnets[].SubnetId)
    export PUBLIC_SUBNETS_ID_B=$(aws ec2 describe-subnets --filters "Name=tag:Name,Values=k8s-us-east-1b-public-subnet" | jq -r .Subnets[].SubnetId)
    export PUBLIC_SUBNETS_ID_C=$(aws ec2 describe-subnets --filters "Name=tag:Name,Values=k8s-us-east-1c-public-subnet" | jq -r .Subnets[].SubnetId)

Prepare the manifest for Network Load Balancer for server application pods. The MEAN code requires client to directly access NLB from public internet. The NLB must be deployed in public subnets. NLB must be internet facing.

    cat > /home/ec2-user/environment/deploy_nlb.yaml <<EOF
    apiVersion: v1
    kind: Service
    metadata:
      namespace: mongodb
      name: server-nlb
      annotations:
        service.beta.kubernetes.io/aws-load-balancer-type: external
        service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
        service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
        service.beta.kubernetes.io/aws-load-balancer-subnets: $PUBLIC_SUBNETS_ID_A, $PUBLIC_SUBNETS_ID_B, $PUBLIC_SUBNETS_ID_C
    
        service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    
    spec:
      type: LoadBalancer
      ports:
        - port: 5200
          protocol: TCP
          targetPort: 5200
          name: server
      selector:
        app: server
    EOF

Deploy Network Load Balancer on public subnets

    kubectl apply -f deploy_nlb.yaml

    kubectl get svc -n mongodb

NAME             TYPE                  CLUSTER-IP           EXTERNAL-IP                                                                                                                        PORT(S)              AGE
server-nlb         LoadBalancer    172.20.103.203        k8s-mongodb-servernl-9c3c0762d8-xxxxxxxxxxxxx.elb.us-east-1.amazonaws.com   5200:30308/TCP   33m
server-service   ClusterIP           172.20.38.49            <none>                                                                                                                                 5200/TCP               42m

Copy the server-nlb External-IP address and paste it to notepad for future use.

Verify NLB target group is healthy Go to EC2  Target Groups and select the target group for NLB. There should be at least one target in health. This will take about 3 minutes. image

Get the URI of NLB Copy the server-nlb External-IP address and paste it to notepad for future use. The URI here is - k8s-mongodb-servernl-9c3c0762d8-xxxxxxxxxx.elb.us-east-1.amazonaws.com

Set the NLB URI for the client

Client uses file ~/environment/MEANStack_with_Atlas_on_Fargate/code/MEANSTACK/partner-meanstack-atlas-fargate/client/src/app/employee.service.ts to get NLB URI. This URI will be sent directly to the client browser. The client browser will use this URI to access the server.

    cd ~/environment/MEANStack_with_Atlas_on_Fargate/code/MEANSTACK/partner-meanstack-atlas-fargate/client/src/app

Edit the file to update the PrivateURL

    nano employee.service.ts



import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable, Subject, tap } from 'rxjs';
import { Employee } from './employee';

@Injectable({
  providedIn: 'root'
})
export class EmployeeService {
   **private url = 'http:// k8s-mongodb-servernl-9c3c0762d8-xxxxxxxxxx.elb.us-east-1.amazonaws.com:5200';**
  /*private url = 'http://<ipaddress of the server>.us-east-1.elasticbeanstalk.com:5200';*/
  private employees$: Subject<Employee[]> = new Subject();

Save the file and change it to the directory

    cd ~/environment/MEANStack_with_Atlas_on_Fargate/code/MEANSTACK/partner-meanstack-atlas-fargate

Set the environment variable for ACCOUNT_ID

    export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

Rebuild the client application and push it to ECR.

    aws ecr get-login-password --region us-east-1| docker login --username AWS --password-stdin ${ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com
    docker context use default
    docker-compose build
    docker-compose push	

Go to ECR and verify that the new client application image is pushed to ECR repository.

Create client deployment manifest.

    cat > /home/ec2-user/environment/deploy_client.yaml <<EOF
    apiVersion: apps/v1
    kind: Deployment 
    metadata:
      name: client-deployment
      namespace: mongodb
    spec:
      selector:
        matchLabels:
          app: client
      template:
        metadata:
          labels:
            app: client
        spec:
          containers:
          - name: client
            image: ${ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com/partner-meanstack-atlas-eks-client:latest # specify your ECR repository
            ports:
            - containerPort: 8080
            resources:
                limits:
                  cpu: 500m
                requests:
                  cpu: 250m
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: client-service
      namespace: mongodb
      labels:
        app: client
    spec:
      selector:
        app: client
      ports:
        - protocol: TCP
          port: 8080 
          targetPort: 8080
      type: NodePort # expose the service as NodePort type so that ALB can use it later.
    EOF

Deploy client application

    cd ~/environment
    kubectl apply -f deploy_client.yaml

Verify the deployment.

    kubectl get pods -n mongodb

NAME                                 READY   STATUS    RESTARTS   AGE
client-deployment-54974f4bb8-ct7wq   1/1     Running   0          25s
Expose the application to internet

To achieve this, we need to install the aws-load-balancer-controller, the controller shall help the k8s cluster to manage the lifecycle of the load balancer via Ingress' API. To explore more, read the official [documentation](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.2/). Because we have used eksctl` to provision necessary IAM policy and service account resources, we can simply install the controller by using Helm. Add the EKS chart repo to helm

    helm repo add eks https://aws.github.io/eks-charts

Install the helm chart if using IAM roles for service accounts. NOTE you need to specify both of the chart values serviceAccount.create=false and serviceAccount.name=aws-load-balancer-controller

    helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=web-host-on-eks --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller

Because we shall deploy the Ingress(ALB) to the public subnets to serve the private application as the single traffic door, we need to grab the public subnets to the template.



    # export public subnets
    export PUBLIC_SUBNETS_ID_A=$(aws ec2 describe-subnets --filters "Name=tag:Name,Values=k8s-us-east-1a-public-subnet" | jq -r .Subnets[].SubnetId)
    export PUBLIC_SUBNETS_ID_B=$(aws ec2 describe-subnets --filters "Name=tag:Name,Values=k8s-us-east-1b-public-subnet" | jq -r .Subnets[].SubnetId)
    export PUBLIC_SUBNETS_ID_C=$(aws ec2 describe-subnets --filters "Name=tag:Name,Values=k8s-us-east-1c-public-subnet" | jq -r .Subnets[].SubnetId)

Create the ingress yaml

    cat > /home/ec2-user/environment/ingress.yaml <<EOF
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      namespace: mongodb
      name: ingress-client
      annotations:
        kubernetes.io/ingress.class: alb
        alb.ingress.kubernetes.io/scheme: internet-facing
        alb.ingress.kubernetes.io/target-type: ip # using IP routing policy of ALB
        alb.ingress.kubernetes.io/subnets: $PUBLIC_SUBNETS_ID_A, $PUBLIC_SUBNETS_ID_B, $PUBLIC_SUBNETS_ID_C # specifying the public subnets id
    spec:
      rules:
        - http:
            paths:
              - path: /
                pathType: Prefix
                backend:
                  service:
                    name: client-service # refer to the service defined in deploy_client.yaml
                    port:
                      number: 8080
    EOF


Deploy ALB

    kubectl apply -f ingress.yaml

ingress.networking.k8s.io/ingress-client created

Check if the Ingress(ALB) is running.

    kubectl get ing -n mongodb

NAME             CLASS    HOSTS   ADDRESS                                                                 PORTS   AGE
ingress-client   <none>   *       k8s-mongodb-ingressc-4af057c13f-xxxxxxxxx.us-east-1.elb.amazonaws.com   80      12s

Go to EC2 --> Load Balancer and wait until ALB is provisioned and the target group passes the health check. Copy the DNS name of ALB and open the browser with this DNS name

### Step5: Testing the Application
Copy the DNS name of ALB and open the browser with this DNS name http://k8s-mongodb-ingressc-xxxxxxxx.us-east-1.elb.amazonaws.com image

<img width="827" alt="image" src="https://github.com/mongodb-partners/MEAN_Stack_With_MongoDB_Atlas_On_Amazon_EKS/assets/101570105/c0046077-50ae-47f9-9496-194e261bd63a">


Login to the MongoDB console and check Database Demo and Collections Employees.

<img width="827" alt="image" src="https://github.com/mongodb-partners/MEAN_Stack_With_MongoDB_Atlas_On_Amazon_EKS/assets/101570105/0cbb664d-b677-493d-a95a-b777c5b9daf9">


Add a new user and delete the user from the application GUI. You should see changes in the MongoDB database.

**Congratulations! You finished the Accelerate Application Modernization with Amazon EKS and MongoDB Atlas lab**

### Step 6 - Clean UP

    cd /home/ec2-user/environment
    kubectl delete -f ingress.yaml
    kubectl delete -f deploy_client.yaml
    kubectl delete -f deploy_nlb.yaml
    kubectl delete -f deploy_server.yaml

    eksctl delete cluster --name web-host-on-eks

Go to the AWS EC2 console Terminate eks-bastion EC2 instance

Go to the AWS VPC console. Delete NAT gateway, delete VPC named k8-vpc.

Go to the AWS ECR console and delete repository for server and client

## Summary
Hope this provides the steps to successfully deploy the containerized application onto AWS EKS. Please share your feedback/queries to partners@mongodb.com

