# Trend App – Production Deployment (Docker, Terraform, EKS, Jenkins CI/CD, Monitoring)

A React application (Trendify) deployed to AWS EKS with a fully automated CI/CD pipeline.

- **GitHub repo:** https://github.com/srisubi1998-ctrl/Trend
- **DockerHub image:** https://hub.docker.com/r/subitcha1998/trend-app
- **Kubernetes LoadBalancer ARN:**
  `arn:aws:elasticloadbalancing:us-east-1:196390795947:loadbalancer/net/a2273f9d63e7a4414a5dfcbecfc713b1/e2062fb7fb750226`
- **Live app (LoadBalancer URL):** http://a2273f9d63e7a4414a5dfcbecfc713b1-e2062fb7fb750226.elb.us-east-1.amazonaws.com

## Architecture

```
Developer → GitHub → (webhook) → Jenkins (EC2)
                                   ├─ Build Docker image
                                   ├─ Push to DockerHub
                                   └─ kubectl apply → AWS EKS → LoadBalancer → Users
                                                       └─ Prometheus + Grafana (monitoring)
```

## Project Structure

```
Trend/
├── dist/                # React production build
├── Dockerfile           # nginx image serving the app on port 3000
├── nginx.conf           # nginx config (listen 3000, React routing)
├── Jenkinsfile          # Declarative CI/CD pipeline
├── k8s/
│   ├── deployment.yaml  # 2 replicas of trend-app
│   └── service.yaml     # LoadBalancer service (NLB)
├── terraform/
│   └── main.tf          # VPC, subnets, IAM roles, EKS cluster + node group
├── screenshots/
├── .gitignore
└── .dockerignore
```

---

## 1. Version Control

The code is pushed to GitHub using the CLI. `.gitignore` and `.dockerignore` are included.

```bash
git init
git add .
git commit -m "Initial commit"
git remote add origin https://github.com/srisubi1998-ctrl/Trend.git
git push -u origin main
```

![GitHub repo](screenshots/github_repo.png)
![GitHub repo updated](screenshots/github_repo_updated.png)

## 2. Docker

The app is packaged with a Dockerfile (nginx serving the built app on port 3000).

```bash
docker build -t trend-app .
docker run -d -p 3000:3000 --name trend-container trend-app
```

![Docker image and container](screenshots/docker-build.png)
![App running on EC2](screenshots/React_App_Live_Deployment_EC2.png)

## 3. DockerHub

The image is pushed to DockerHub as `subitcha1998/trend-app`.

```bash
docker login
docker tag trend-app subitcha1998/trend-app:latest
docker push subitcha1998/trend-app:latest
```

![DockerHub repository](screenshots/dockerhub-repo.png)

## 4. Terraform (Infrastructure as Code)

`terraform/main.tf` creates:
- VPC, internet gateway, 2 public subnets, route table
- IAM roles for the EKS cluster and worker nodes
- EKS cluster (`trend-cluster`) and a managed node group (2 x t3.medium)

```bash
cd terraform
terraform init
terraform plan
terraform apply
```

![Terraform plan](screenshots/terraform-plan.png)
![Terraform apply](screenshots/terraform-apply.png)
![VPC console](screenshots/vpc-console.png)

## 5. AWS setup and Jenkins server

An EC2 instance (`Trend-DevOps-Server`) runs Jenkins. An IAM user with CLI access is used by Terraform and Jenkins.

![IAM permissions](screenshots/AdministratorAccess.png)
![AWS CLI version](screenshots/aws-cli-version.png)
![AWS identity](screenshots/aws-identity.png)
![EC2 instance](screenshots/AWS_EC2_Instance_Running.png)

## 6. Jenkins

Jenkins is installed on the EC2 server with the Git, Docker, Kubernetes and Pipeline plugins. DockerHub credentials are stored in Jenkins credentials.

![Jenkins running](screenshots/Jenkins_Service_Status.png)
![Jenkins unlock](screenshots/Jenkins_Unlock_Page.png)
![Jenkins dashboard](screenshots/Jenkins_Dashboard.png)
![Jenkins credentials](screenshots/Jenkins_Credentials_Management_Page.png)
![Jenkins pipeline configuration](screenshots/Jenkins_Pipeline_Configuration_Screen.png)
![Jenkins can reach EKS](screenshots/jenkins-kubectl-nodes.png)

## 7. Kubernetes on AWS EKS

The EKS cluster is created with Terraform. The app is deployed using `k8s/deployment.yaml` and `k8s/service.yaml`.

```bash
aws eks update-kubeconfig --region us-east-1 --name trend-cluster
kubectl get nodes
kubectl apply -f k8s/
kubectl get pods
kubectl get svc trend-service
```

![EKS cluster active](screenshots/eks-cluster-active.png)
![kubectl nodes](screenshots/kubectl-nodes.png)
![k8s YAML files](screenshots/k8s-yaml-files.png)
![Pods and service](screenshots/kubectl-pods-svc.png)
![App on LoadBalancer](screenshots/app-loadbalancer.png)
![LoadBalancer ARN](screenshots/loadbalancer-arn.png)

## 8. CI/CD Pipeline (Jenkinsfile)

The declarative pipeline has 4 stages:

1. **Checkout Code** – pulls the latest code from GitHub
2. **Build Docker Image** – builds `subitcha1998/trend-app:latest`
3. **Push to DockerHub** – logs in using Jenkins credentials and pushes the image
4. **Deploy to EKS** – runs `kubectl apply` and restarts the deployment

![Pipeline stage view](screenshots/Jenkins_Stage_View.png)
![Console output](screenshots/Jenkins_Console_Output.png)
![Build history](screenshots/Jenkins_Build_History_and_Execution_Details.png)

## 9. GitHub Webhook (auto build on every commit)

Webhook URL: `http://<jenkins-ip>:8080/github-webhook/` (push events, JSON).
In Jenkins, "GitHub hook trigger for GITScm polling" is enabled, so every push starts a build.

![Webhook green tick](screenshots/webhook-green-tick.png)
![Build started by GitHub push](screenshots/auto-triggered-build.png)

## 10. Monitoring (Prometheus + Grafana)

Installed on the cluster with Helm (open source):

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
kubectl create namespace monitoring
helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring \
  --set grafana.service.type=LoadBalancer
```

![Monitoring pods](screenshots/monitoring-pods.png)
![Grafana namespace pods dashboard](screenshots/grafana-dashboard.png)
![Grafana cluster dashboard](screenshots/grafana-cluster.png)

---

## How to reproduce

1. Run `terraform apply` in `terraform/` to create the VPC, IAM roles and EKS cluster.
2. Install Jenkins on an EC2 server and add the DockerHub credentials (`dockerhub-credentials`).
3. Give Jenkins AWS access and connect it to the cluster with `aws eks update-kubeconfig`.
4. Create a Pipeline job pointing at this repo and add the GitHub webhook.
5. Push a commit. Jenkins builds, pushes and deploys automatically.
6. Open the LoadBalancer URL to see the app.
