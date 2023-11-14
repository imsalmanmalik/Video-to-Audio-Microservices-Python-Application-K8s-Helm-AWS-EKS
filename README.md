## Video to Audio Microservices Python Application onto AWS EKS 
Converting mp4 videos to mp3 in a microservices architecture.

## Technology Stack
This project leverages a variety of tools, platforms, and skills. Below is a comprehensive list:

1) **Programming Languages & Frameworks**:
* Python: Primary programming language used for microservices.
* Flask: Web framework for building microservices in Python.

2) **Containerization & Orchestration**:
* Docker: Used for containerizing the microservices.
* Kubernetes: Orchestration tool for managing containerized applications.
* Helm: Package manager for Kubernetes, used for managing Kubernetes applications.

3) **Database & Messaging**:
* PostgreSQL: Relational database used for data storage.
* MongoDB: NoSQL database used for flexible data storage.
* RabbitMQ: Messaging broker for handling communication between services.
  
4) **Cloud & Infrastructure**:
* AWS (Amazon Web Services): Cloud platform for hosting the application.
* Amazon EKS (Elastic Kubernetes Service): Managed Kubernetes service on AWS.
* AWS CloudWatch: Monitoring and observability service used for logging and metrics.
  
5) **Security & Authentication**:
* JWT (JSON Web Tokens): Used for secure authentication in the application.
* Secrets Management: Kubernetes secrets used for managing sensitive data.

## Architecture

<p align="center">
  <img src="./images/ProjectArchitecture.png" width="600" title="Architecture" alt="Architecture">
  </p>

### Introduction

This document provides a step-by-step guide for deploying a Python-based microservice application on AWS Elastic Kubernetes Service (EKS). The application comprises four major microservices: `auth-server`, `converter-module`, `gateway server` and `notification-server`.

### Prerequisites

Before you begin, ensure that the following prerequisites are met:

1. **Create an AWS Account:** If you do not have an AWS account, create one by following the steps [here](https://docs.aws.amazon.com/streams/latest/dev/setting-up.html).

2. **Install Helm:** Helm is a Kubernetes package manager. Install Helm by following the instructions provided [here](https://helm.sh/docs/intro/install/).

3. **Python:** Ensure that Python is installed on your system. You can download it from the [official Python website](https://www.python.org/downloads/).

4. **AWS CLI:** Install the AWS Command Line Interface (CLI) following the official [installation guide](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html).

5. **Install kubectl:** Install the latest stable version of `kubectl` on your system. You can find installation instructions [here](https://kubernetes.io/docs/tasks/tools/).

6. **Databases:** Set up PostgreSQL and MongoDB for your application.

### Steps to follow

#### Cluster Creation

1. **Log in to AWS Console:**
   - Access the AWS Management Console with your AWS account credentials.

2. **Create eksCluster IAM Role**
   - Follow the steps mentioned in [this](https://docs.aws.amazon.com/eks/latest/userguide/service_IAM_role.html) documentation using root user
   - After creating it will look like this:

   <p align="center">
  <img src="./images/ekscluster_role.png" width="600" title="ekscluster_role" alt="ekscluster_role">
  </p>

   - Please attach `AmazonEKS_CNI_Policy` explicitly if it is not attached by default

3. **Create Node Role - AmazonEKSNodeRole**
   - Follow the steps mentioned in [this](https://docs.aws.amazon.com/eks/latest/userguide/create-node-role.html#create-worker-node-role) documentation using root user
   - Please note that you do NOT need to configure any VPC CNI policy mentioned after step 5.e under Creating the Amazon EKS node IAM role
   - Simply attach the following policies to your role once you have created `AmazonEKS_CNI_Policy` , `AmazonEBSCSIDriverPolicy` , `AmazonEC2ContainerRegistryReadOnly` , `CloudWatchAgentServerPolicy`(for observability & logging at cluster and application level)
   - Your AmazonEKSNodeRole will look like this: 

<p align="center">
  <img src="./images/eksnode_role_policies.png" width="600" title="Node_IAM" alt="Node_IAM">
  </p>

4. **Open EKS Dashboard:**
   - Navigate to the Amazon EKS service from the AWS Console dashboard.

5. **Create EKS Cluster:**
   - Click "Create cluster."
   - Choose a name for your cluster.
   - Configure networking settings (VPC, subnets).
   - Choose the `eksCluster` IAM role that was created above
   - Review and create the cluster.

6. **Cluster Creation:**
   - Wait for the cluster to provision, which may take several minutes.

7. **Cluster Ready:**
   - Once the cluster status shows as "Active," you can now create node groups.

#### Node Group Creation

1. In the "Compute" section, click on "Add node group."

2. Choose the AMI (default), instance type (e.g., t3a.xlarge), and the number of nodes(1 node is sufficient for testing out this project).

3. Click "Create node group."

#### Adding inbound rules in Security Group of Nodes

**NOTE:** Ensure that all the necessary ports are open in the node security group.

<p align="center">
  <img src="./images/inbound_rules_sg.png" width="600" title="Inbound_rules_sg" alt="Inbound_rules_sg">
  </p>

#### Enable EBS CSI and CloudWatch Observability Addon
1. enable addon `ebs csi` this is for enabling pvc's once cluster is created.
2. enable `Amazon CloudWatch Observability`(For installing the CloudWatch agent to enable Container Insights within the cluster).

<p align="center">
  <img src="./images/eks-add-ons.png" width="600" title="ebs_addon" alt="ebs_addon">
  </p>

#### Deploying your application on EKS Cluster

1. Clone the code from this repository.

2. Set the cluster context:
   ```
   aws eks update-kubeconfig --name <cluster_name> --region <aws_region>
   ```

### Commands

Here are some essential Kubernetes commands for managing your deployment:


### MongoDB

To install MongoDB, set the database username and password in `values.yaml`, then navigate to the MongoDB Helm chart folder and run:

```
cd Helm_charts/MongoDB
helm install mongo .
```

Connect to the MongoDB instance using:

```
mongosh 'mongodb://<username>:<pwd>@<nodeip>:30005/mp3s?authSource=admin'
```
**Note**: `nodeip` can be obtained by navigating to EKS >Clusters >Your cluster >Compute >Node Groups >Node >InstanceId >PublicIpAddress(copy)

### PostgreSQL

Set the database username and password in `values.yaml`. Install PostgreSQL from the PostgreSQL Helm chart folder and initialize it with the queries in `init.sql`.

```
cd ..
cd Postgres
helm install postgres .
```

Connect to the Postgres database and copy all the queries from the "init.sql" file.
```
psql 'postgres://<username>:<pwd>@<nodeip>:30003/authdb'
```

### RabbitMQ

Deploy RabbitMQ by running:

```
helm install rabbitmq .
```

Ensure you have created two queues in RabbitMQ named `mp3` and `video`. To create queues, visit `<nodeIp>:30004>` and use default username `guest` and password `guest`

<p align="center">
  <img src="./images/rabbitmq.png" width="600" title="ebs_addon" alt="ebs_addon">
  </p>

**NOTE:** Ensure that all the necessary ports are open in the node security group.

### Building and Pushing Docker Images
You can create your own Docker images from the Dockerfiles present for each microservice and push them to Docker Hub, or you can use my existing images. Below are the microservices available and the commands to build and push their Docker images:

```
# Build the Docker image
docker build -t <your-docker-hub-username>/<microservice-name>:latest ./src/<microservice-name>

# Push the image to Docker Hub
docker push <your-docker-hub-username>/<microservice-name>:latest
```

### Apply the manifest file for each microservice:

- **Auth Service:**
  ```
  cd auth-service/manifest
  kubectl apply -f .
  ```

- **Gateway Service:**
  ```
  cd gateway-service/manifest
  kubectl apply -f .
  ```

- **Converter Service:**
  ```
  cd converter-service/manifest
  kubectl apply -f .
  ```

- **Notification Service:**
  ```
  cd notification-service/manifest
  kubectl apply -f .
  ```

### Application Validation

After deploying the microservices, verify the status of all components by running:

```
kubectl get all
```

### Notification Configuration

For configuring email notifications and two-factor authentication (2FA), follow these steps:

1. Go to your Gmail account and click on your profile.

2. Click on "Manage Your Google Account."

3. Navigate to the "Security" tab on the left side panel.

4. Enable "2-Step Verification."

5. Search for the application-specific passwords. You will find it in the settings.

6. Click on "Other" and provide your name.

7. Click on "Generate" and copy the generated password.

8. Paste this generated password in `notification-service/manifest/secret.yaml` along with your email.

Run the application through the following API calls:

# API Definition

- **Login Endpoint**
  ```http request
  POST http://nodeIP:30002/login
  ```
  
  ```console
  curl -X POST http://nodeIP:30002/login -u <email>:<password>
  ``` 
  Expected output: JWT Token
  
Once the login API endpoint is called, a JWT token is produced. Users can decode this token using the online base64 decoder [here](https://www.base64decode.org/)


- **Upload Endpoint**
  ```http request
  POST http://nodeIP:30002/upload
  ```

  ```console
   curl -X POST -F 'file=@./video.mp4' -H 'Authorization: Bearer <JWT Token>' http://nodeIP:30002/upload
  ``` 
  
  Check if you received the File ID(fid) on your email and copy the string for the next step.
  
  *Note*: The video upload is going to take some time since it is a long 5 min mp4 video which is a video recording of one of my projects 'MLOps-end-to-end-project'. This is the [link](https://github.com/imsalmanmalik/MLOps-end-to-end-project) to the MLOps project if anyone is interested. Feel free to replace it with a shorter video. 

- **Download Endpoint**
  ```http request
  GET http://nodeIP:30002/download?fid=<Generated file identifier>
  ```
  ```console
   curl --output video.mp3 -X GET -H 'Authorization: Bearer <JWT Token>' "http://nodeIP:30002/download?fid=<Generated fid>"
  ``` 
**Note**: Replace with the `fid` recieved on your personal email along with the `JWT Token` recieved earlier after the Login Endpoint was called and the `nodeIP` of your EKS Node Group worker instance.

## Destroying the Infrastructure

To clean up the infrastructure, follow these steps:

1. **Delete the Node Group:** Delete the node group associated with your EKS cluster.

2. **Delete the EKS Cluster:** Once the nodes are deleted, you can proceed to delete the EKS cluster itself.
