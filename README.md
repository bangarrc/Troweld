Create 1 EC2 Instance= 3-tier-hq
git clone

@Frontend image creation

Open repo, go to frontend & create dockerfile
vim Dockerfile
FROM node:14
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
CMD ["npm","start"]
sudo apt-get update
sudo apt install docker.io
docker ps-it will not run,access denied
whoami
echo $USER
sudo chown $USER /var/run/docker.sock
docker ps-Now it will work
To create image
docker build -t three-tier-frontend .
To create container which runs on 3000 port
docker run -d -p 3000:3000 three-tier-frontend
open port 3000 in security group & check if it is working or not

@AWS CLI Installation (to push image to ECR) 

curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip
unzip awscliv2.zip
sudo ./aws/install -i /usr/local/aws-cli -b /usr/local/bin --update
aws configure

@IAM Configuration

Create a user eks-admin with AdministratorAccess.
Generate Security Credentials: Access Key and Secret Access Key.
Access key-AKIAQ3EGVZCNCHI2OWFJ
Secret access key-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

@ECR image push

Repo name= three-tier-frontend
commands are available in repo= push commands button (Run this commands in frontend dir)

@Backend image creation
Dockerfile

FROM node:14
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
CMD ["node","index.js"]

@ECR image push

Repo name=> three-tier-backend
commands are available in repo= push commands button (Run this commands in frontend dir)

@Install kubectl==>   (This helps to control or manage kubernetes cluster)

curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
chmod +x ./kubectl  (+x= to execute kubctl commands)
sudo mv ./kubectl /usr/local/bin   (if we move >/kubectl to bin then we don't need to use ./ to run command, directly you can use kubectl)
kubectl version --short --client

@Install eksctl==>   (This helps to create kubernetes cluster on EKS Service)  

curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp  (--silent is used to run it in background-not shows logs)
sudo mv /tmp/eksctl /usr/local/bin
eksctl version

@Setup EKS Cluster

eksctl create cluster --name three-tier-cluster --region us-west-2 --node-type t2.medium --nodes-min 2 --nodes-max 2
aws eks update-kubeconfig --region us-west-2 --name three-tier-cluster  
==>(This command bind kubectl with EKS cluster so that on running command "kubectl get nodes" it will get the details of that mentioned cluster only)==>for multiple clusters)
kubectl get nodes

echo 'rushikesh' | base64    ==>This used to create username & password for database (kubernetes manifests file==>secrets.yaml file)
echo 'cnVzaGlrZXNoCg==' | base64 --decode
echo -n "password123" | base64 -i -     uname=dXNlcm5hbWU=   passwd=cGFzc3dvcmQxMjM=

@Run Manifests

1)Database

kubectl create namespace three-tier or workshop
kubectl apply -f deployment.yaml(file name)
kubectl get deployment -n namespace(three-tier or workshop)
kubectl apply -f secrets.yaml
kubectl get pods -n three-tier
kubectl apply -f service.yaml
kubectl get svc -n three-tier
kubectl apply -f .
kubectl delete -f .

2)Backend

kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl get pods -n three-tier
kubectl logs pod_name(backend pod name) -n namespace(three-tier) =====> It will connect backend with database

3)Frontend

kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl get pods -n three-tier


To associate ALB with EKS cluster we need to write following commands. (i.e. To give access of EKS cluster to user, we have to attach some policies through this commands) 

@Install AWS Load Balancer
cd
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json --->To Download IAM Policy
aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json --->To create IAM Policy

eksctl utils associate-iam-oidc-provider --region=us-west-2 --cluster=three-tier-cluster --approve

To create IAM Role:following command
eksctl create iamserviceaccount --cluster=three-tier-cluster --namespace=kube-system --name=aws-load-balancer-controller --role-name AmazonEKSLoadBalancerControllerRole --attach-policy-arn=arn:aws:iam::626072240565:policy/AWSLoadBalancerControllerIAMPolicy --approve --region=us-west-2  ----------->It will create service account of load balancer
                                                                     (Account No.)
@Helm==> We are going to install ingress controller through helm

===> Helm is a package manager.The kubernetes manifests files,charts are present in helm.(This manifests files,charts are nothing but to run load balancer) we just need to install helm.

@Deploy AWS Load Balancer Controller
sudo snap install helm --classic
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks
helm repo list =====>it will show repo
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=my-cluster --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller
kubectl get deployment -n kube-system aws-load-balancer-controller

@To create Ingress controller
kubectl apply -f full_stack_lb.yaml
kubectl get ing -n three-tier
==>It will show address, copy the address & go to godaddy & associate this address with your domain name. So it will do internal routing of clusters

steps==>Go daddy==>Domain portfolio==>DNS==>Add new record==>Type=C Name==>Name=your domain name==>value=paste address of ingress==>TTL=600

@delete the EKS cluster

eksctl delete cluster --name three-tier-cluster --region us-west-2
