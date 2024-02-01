Create the EKS cluster with terraform.
Prerequisite
1. AWS Cli install and configured
2. Terraform install on our local system.
Then, create the infrastruture of EKS Cluster to create the EKS through terraform.
provider "aws" {                                                                                                            
  region = "us-east-1"                                                                                                      
}
resource "aws_eks_cluster" "my_cluster" {                                                                                   
  name     = "my-eks-cluster"
  role_arn = aws_iam_role.eks_cluster_role.arn

vpc_config {
    subnet_ids         = ["subnet-003f0648235fa127a", "subnet-04b18b98fc5c485f5"]
    security_group_ids = ["sg-0e7300f86f7f86589"]
    endpoint_public_access = true
    endpoint_private_access = true
  }
}

resource "aws_eks_node_group" "my_node_group" {
  cluster_name    = aws_eks_cluster.my_cluster.name                                                                           node_group_name = "my-node-group"
  node_role_arn   = aws_iam_role.eks_node_group_role.arn
  subnet_ids      = ["subnet-003f0648235fa127a", "subnet-04b18b98fc5c485f5"]

scaling_config {                                                        
    desired_size = 2
    max_size     = 3
    min_size     = 1
  }
}

resource "aws_iam_role" "eks_cluster_role" {
  name = "eks_cluster_role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Effect    = "Allow",
        Principal = {
          Service = "eks.amazonaws.com"
        },
        Action    = "sts:AssumeRole"
      }
    ]
  })
}

resource "aws_iam_role" "eks_node_group_role" {
  name = "eks_node_group_role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [
      {
        Effect    = "Allow",
        Principal = {
          Service = "ec2.amazonaws.com"
        },
        Action    = "sts:AssumeRole"
      }
    ]
  })
}
  

EKS cluster created successfully.
So, in this terraform file we've defined,
•	Provider Block: Specifies the AWS provider and the desired region.
•	AWS EKS Cluster Resource: Defines the EKS cluster, including its name, VPC configuration, and associated IAM role.
•	AWS EKS Node Group Resource: Specifies the worker nodes for the cluster, including the cluster name, node group name, 
    and scaling configuration.
•	AWS IAM Role Resources: Defines IAM roles for the EKS cluster and node group.

Now, Deploy the nodejs application with the high availability on EKSusing Helm chart and expose it with the networl Load Balancer.
Prerequisites:-
1. Helm installed on our Local Machine
2. Install Docker
3. Then, Dockerized the nodejs applicationand and packaged to be deploy on EKS.
Install helm
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm

Helm Installed successfully.

Then, Docker installed.
apt install docker.io -y

Dockerized nodejs application and pushed into Image Registry. i.e. Dockerhub.
docker build -t imagejs . 

To push image into dockerhub we need to tag image with Repository.
docker tag imagejs prodrepo/imagejs
docker push prodrepo/imagejs

Then, Create nodejs chart by helm
helm create nodejs

•	In values.yaml file oh nodejs-chart we do some change i.e. repository of image, tags, and the port no. of service. So, as per the changes, the image will be changed.
replicaCount: 3
image:
    repository:imagejs
tag: “latest”
port: 3000

Now, inside the template of nodejs chart there is deployment.yml and service.yml. We modify that manifests.
i.e.
Deployment Configuration (deployment.yaml):-
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nodejs-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-nodejs-app
  template:
    metadata:
      labels:
        app: my-nodejs-app
    spec:
      containers:
      - name: my-nodejs-app
        image: imagejs:latest
        ports:
        - containerPort: 3000

Service Configuration (service.yaml):-
apiVersion: v1
kind: Service
metadata:
  name: my-nodejs-app
spec:
  type: LoadBalancer
  selector:
    app: my-nodejs-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000

Deployment Steps:-
• Package our helm chart
helm package my-nodejs-app/

• Then, we deploy the helm Chart.
Install the helm chart on our EKS Cluster
helm install my-nodejs-app ./my-nodejs-app-0.1.0.tgz

• Verify the Deployment
kubectl get pods
kubectl get services

Now, we can access our application by using NLB’s DNS name or thr ip-address.





