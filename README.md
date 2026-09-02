
# 🚀 Brain-Tasks-App — React App CI/CD Deployment on AWS EKS
<p align="center">
  <img src="https://img.shields.io/badge/React-18-blue?logo=react" alt="React" />
  <img src="https://img.shields.io/badge/Docker-Container-2496ED?logo=docker&logoColor=white" alt="Docker" />
  <img src="https://img.shields.io/badge/DockerHub-Registry-2496ED?logo=docker&logoColor=white" alt="DockerHub" />
  <img src="https://img.shields.io/badge/Jenkins-CI%2FCD-D24939?logo=jenkins&logoColor=white" alt="Jenkins" />
  <img src="https://img.shields.io/badge/Kubernetes-AWS%20EKS-326CE5?logo=kubernetes&logoColor=white" alt="Kubernetes" />
  <img src="https://img.shields.io/badge/Monitoring-CloudWatch-FF9900?logo=amazonaws&logoColor=white" alt="Monitoring" />
</p>
<p align="center">
  Taking a React application from source code to a fully automated, production-ready deployment — using Docker, Jenkins, DockerHub, and AWS EKS.
</p>
## 📑 Table of Contents
- [Overview](#-overview)
- [Tech Stack](#-tech-stack)
- [Project Structure](#-project-structure)
- [Prerequisites](#-prerequisites)
- [Setup Instructions](#️-setup-instructions)
- [Pipeline Explanation](#-pipeline-explanation)
- [Live Deployment](#-live-deployment)
- [Screenshot Evidence](#-screenshot-evidence)
- [License](#-license)
## 🧭 Overview
This project takes a React application from source code to a fully automated production deployment:
```
GitHub push → Jenkins (build & test) → Docker image → DockerHub → Kubernetes (AWS EKS) → LoadBalancer → Live application
                                                                          ↑
                                                              CloudWatch (logs & monitoring)
```
Every push to `main` automatically triggers Jenkins to build a new Docker image, push it to DockerHub, and roll it out to the Kubernetes cluster — no manual steps required after the initial setup.
## 🛠 Tech Stack
| Layer | Tool |
|---|---|
| Application | React |
| Containerization | Docker |
| CI/CD | Jenkins |
| Image Registry | DockerHub |
| Orchestration | Kubernetes on AWS EKS |
| Monitoring | AWS CloudWatch (Container Insights) |
## 📁 Project Structure
```
Brain-Tasks-App/
├── Dockerfile              # Multi-stage build for the React app
├── .dockerignore
├── .gitignore
├── Jenkinsfile              # Declarative CI/CD pipeline
├── k8s/
│   ├── deployment.yaml      # Kubernetes Deployment (2 replicas)
│   └── service.yaml         # Kubernetes LoadBalancer Service
├── docs/
│   └── screenshots/         # Screenshots referenced below
└── README.md
```
## ✅ Prerequisites
- GitHub, AWS, and DockerHub accounts
- Installed locally:
  - git
  - node (v18+)
  - docker
  - aws-cli
  - kubectl
  - eksctl
## ⚙️ Setup Instructions
### 1. Clone and run locally
```bash
git clone https://github.com/Roshini31412/Brain-Tasks-App.git
cd Brain-Tasks-App
npm install
npm start          # runs on http://localhost:3000
```
### 2. Build and test the Docker image
```bash
docker build -t brain-tasks-app:latest .
docker run -d -p 3000:3000 --name brain-tasks-app-test brain-tasks-app:latest
```
### 3. Push the image to DockerHub
```bash
docker login
docker tag brain-tasks-app:latest roshini31/brain-tasks-app:latest
docker push roshini31/brain-tasks-app:latest
```
### 4. Configure Jenkins
1. Install Jenkins (on an EC2 instance or your preferred host) and visit `http://<jenkins-host>:8080` to unlock it.
2. Install plugins: `Docker Pipeline`, `Git`, `Kubernetes CLI`, `Pipeline`, `Pipeline: AWS Steps`.
3. Add credentials under **Manage Jenkins → Credentials**:
   - `dockerhub-creds` — DockerHub username/token
   - `aws-creds` — AWS access key/secret (or attach an IAM role to the Jenkins host)
4. Add a GitHub webhook pointing at `http://<jenkins-host>:8080/github-webhook/` (a tunnel such as ngrok works well for a Jenkins host that isn't publicly reachable).
### 5. Create the EKS cluster
```bash
eksctl create cluster --name brain-tasks-cluster --region ap-south-1 \
  --nodegroup-name brain-tasks-nodes --node-type t3.medium \
  --nodes 2 --nodes-min 1 --nodes-max 3 --managed
aws eks --region ap-south-1 update-kubeconfig --name brain-tasks-cluster
kubectl get nodes
```
### 6. Deploy to Kubernetes
```bash
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl get svc brain-tasks-app-service   # note the EXTERNAL-IP
```
### 7. Set up the Jenkins pipeline
Create a Pipeline job in Jenkins pointing at this repo's `Jenkinsfile`, with the GitHub webhook trigger enabled.
From this point on, every `git push` to `main` automatically builds, pushes, and deploys the app.
### 8. Monitoring
```bash
aws eks update-cluster-config \
  --name brain-tasks-cluster \
  --region ap-south-1 \
  --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}'
kubectl logs -l app=brain-tasks-app --tail=100 -f
```
Build and deploy logs are available in **Jenkins → job → build number → Console Output**; cluster, node, and application logs stream to **CloudWatch Logs** via Container Insights (fluent-bit).
## 🔄 Pipeline Explanation
The `Jenkinsfile` defines a declarative pipeline with four stages:
| Stage | Description |
|---|---|
| 1. Checkout | Pulls the latest code from GitHub |
| 2. Build Docker Image | Builds the app into a Docker image |
| 3. Push to DockerHub | Logs in and pushes the image with the build-number and `latest` tags |
| 4. Deploy to Kubernetes | Applies the manifests in `k8s/` and rolls out the new image on the EKS cluster |
The pipeline is triggered automatically by a GitHub webhook on every push to `main`.
```groovy
pipeline {
    agent any
    environment {
        DOCKERHUB_CREDS = credentials('dockerhub-creds')
        IMAGE_NAME       = "roshini31/brain-tasks-app"
        IMAGE_TAG        = "${env.BUILD_NUMBER}"
        AWS_REGION       = "ap-south-1"
        CLUSTER_NAME     = "brain-tasks-cluster"
    }
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/Roshini31412/Brain-Tasks-App.git'
            }
        }
        stage('Docker Build') {
            steps {
                sh 'docker build -t $IMAGE_NAME:$IMAGE_TAG -t $IMAGE_NAME:latest .'
            }
        }
        stage('Docker Push') {
            steps {
                sh 'echo $DOCKERHUB_CREDS_PSW | docker login -u $DOCKERHUB_CREDS_USR --password-stdin'
                sh 'docker push $IMAGE_NAME:$IMAGE_TAG'
                sh 'docker push $IMAGE_NAME:latest'
            }
        }
        stage('Deploy to EKS') {
            steps {
                withAWS(credentials: 'aws-creds', region: "${AWS_REGION}") {
                    sh 'aws eks update-kubeconfig --name $CLUSTER_NAME --region $AWS_REGION'
                    sh 'kubectl apply -f k8s/deployment.yaml'
                    sh 'kubectl apply -f k8s/service.yaml'
                    sh 'kubectl rollout status deployment/brain-tasks-app'
                }
            }
        }
    }
    post {
        always {
            sh 'docker logout'
        }
    }
}
```
## 🌐 Live Deployment
| Item | Value |
|---|---|
| Application URL | `http://a84fd2b0b0d224ff6bade7608c8df2d3-1141925910.ap-south-1.elb.amazonaws.com` |
| AWS Region | `ap-south-1` |
| EKS Cluster | `brain-tasks-cluster` |
> The LoadBalancer's external hostname changes if the Service is recreated — check `kubectl get svc brain-tasks-app-service` for the current value.
## 📸 Screenshot Evidence
Each stage below is documented end-to-end with real screenshots from the deployment.
### 01 — Clone and Run Locally
The React app served locally on port 3000.
![Local serve terminal](docs/screenshots/01-local-serve-terminal.png)
*Terminal running the local server and confirming it's serving on `localhost:3000`.*
![App running in browser](docs/screenshots/02-app-running-browser.png)
*Browser showing the Brain Tasks app loaded at `localhost:3000`.*
### 02 — Dockerize and Test the Image
Containerizing the app and verifying it runs from the image.
![Docker container running](docs/screenshots/03-docker-ps-container-running.png)
*`docker ps` showing the `brain-tasks-app-test` container `Up` with port 3000 mapped.*
![App served from container](docs/screenshots/04-app-served-from-container.png)
*Browser showing the app served from the running Docker container.*
### 03 — Push Image to DockerHub
Verifying the DockerHub registry integration.
![Docker push output](docs/screenshots/05-docker-push-output.png)
*`docker push` output showing the image layers pushed to `roshini31/brain-tasks-app`.*
![DockerHub repository](docs/screenshots/06-dockerhub-repository.png)
*DockerHub repository page confirming the `latest` tag was pushed successfully.*
### 04 — EKS Cluster and Kubernetes Deployment
Standing up the cluster and rolling out the app.
![EKS cluster created](docs/screenshots/07-eks-cluster-created.png)
*`eksctl` output confirming the `brain-tasks-cluster` is ready in `ap-south-1`.*
![kubectl get nodes](docs/screenshots/08-kubectl-get-nodes.png)
*`kubectl get nodes` showing all worker nodes in `Ready` status.*
![Manifests applied, pods and service](docs/screenshots/09-kubectl-apply-deploy-pods-svc.png)
*`kubectl apply -f k8s/` followed by `kubectl get pods` (both replicas `Running`) and `kubectl get svc` showing the `LoadBalancer` service with its `EXTERNAL-IP`.*
![App live on LoadBalancer](docs/screenshots/10-app-live-on-loadbalancer.png)
*Browser showing the app served directly from the EKS LoadBalancer URL.*
### 05 — CI/CD Pipeline Setup
Proving the Jenkins pipeline automation, end to end.
![Jenkinsfile committed and pushed](docs/screenshots/11-git-push-jenkinsfile.png)
*Adding the `Jenkinsfile`, committing, and pushing it to `main`.*
![Jenkins pipeline configuration](docs/screenshots/12-jenkins-pipeline-config.png)
*Jenkins job configured with the GitHub repository URL and credentials.*
![Jenkins build success](docs/screenshots/13-jenkins-build-success.png)
*A successful pipeline run, built from the latest commit on `main`.*
![GitHub webhook delivery](docs/screenshots/14-github-webhook-delivery.png)
*GitHub → Settings → Webhooks showing a successful (`200`) delivery to Jenkins.*
### 06 — Monitoring and Logs
Application health, cluster logs, and observability via CloudWatch.
![CloudWatch namespace applied](docs/screenshots/15-cloudwatch-namespace-apply.png)
*Applying the CloudWatch Container Insights manifest to the cluster.*
![fluent-bit pods running](docs/screenshots/16-fluent-bit-pods-running.png)
*`fluent-bit` pods running in the `amazon-cloudwatch` namespace, shipping logs to CloudWatch.*
![CloudWatch log group](docs/screenshots/17-cloudwatch-log-group.png)
*CloudWatch log group for the EKS cluster's control-plane and node logs.*
![kube-apiserver logs](docs/screenshots/18-cloudwatch-kube-apiserver-logs.png)
*Live `kube-apiserver` log events streaming into CloudWatch.*
![Container Insights host logs](docs/screenshots/19-cloudwatch-container-insights-host.png)
*Container Insights host-level logs for an EKS worker node.*
![Application logs](docs/screenshots/20-cloudwatch-application-logs.png)
*Application log stream showing the container accepting connections and serving requests.*
![Jenkins console log](docs/screenshots/21-jenkins-console-log-webhook-events.png)
*Jenkins service console log confirming repeated webhook `PushEvent`s from GitHub triggering builds.*
![EKS control-plane dashboard](docs/screenshots/22-eks-control-plane-dashboard.png)
*EKS control-plane scaling dashboard — API request concurrency, pod scheduling rate, and database size.*
![EKS observability dashboard](docs/screenshots/23-eks-observability-dashboard.png)
*Full EKS observability dashboard — request rates, latency, pending pods, and scheduler activity.*

