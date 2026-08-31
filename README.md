🚀 Brain-Tasks-App — React App CI/CD Deployment on AWS EKS
<p align="center"> <img src="https://img.shields.io/badge/React-18-blue?logo=react" alt="React" /> <img src="https://img.shields.io/badge/Docker-Container-2496ED?logo=docker&logoColor=white" alt="Docker" /> <img src="https://img.shields.io/badge/DockerHub-Registry-2496ED?logo=docker&logoColor=white" alt="DockerHub" /> <img src="https://img.shields.io/badge/Jenkins-CI%2FCD-D24939?logo=jenkins&logoColor=white" alt="Jenkins" /> <img src="https://img.shields.io/badge/Kubernetes-AWS%20EKS-326CE5?logo=kubernetes&logoColor=white" alt="Kubernetes" /> <img src="https://img.shields.io/badge/Monitoring-CloudWatch-FF9900?logo=amazonaws&logoColor=white" alt="Monitoring" /> </p> <p align="center"> Taking a React application from source code to a fully automated, production-ready deployment — using Docker, Jenkins, DockerHub, and AWS EKS. </p> 

📑 Table of Contents
	• Overview
	• Tech Stack
	• Project Structure
	• Prerequisites
	• Setup Instructions
	• Pipeline Explanation
	• Live Deployment
	• Screenshot Evidence
	• License

🧭 Overview
This project takes a React application from source code to a fully automated production deployment:
GitHub push → Jenkins (build & test) → Docker image → DockerHub → Kubernetes (AWS EKS) → LoadBalancer → Live application
                                                                          ↑
                                                              CloudWatch (logs & monitoring)

Every push to main automatically triggers Jenkins to build a new Docker image, push it to DockerHub, and roll it out to the Kubernetes cluster — no manual steps required after the initial setup.

🛠 Tech Stack
Layer	Tool
Application	React
Containerization	Docker
CI/CD	Jenkins
Image Registry	DockerHub
Orchestration	Kubernetes on AWS EKS
Monitoring	AWS CloudWatch

📁 Project Structure
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


✅ Prerequisites
	• GitHub, AWS, and DockerHub accounts
	• Installed locally: 
		○ git
		○ node (v18+)
		○ docker
		○ aws-cli
		○ kubectl
		○ eksctl

⚙️ Setup Instructions
1. Clone and run locally
git clone https://github.com/Vennilavanguvi/Brain-Tasks-App.git
cd Brain-Tasks-App
npm install
npm start          # runs on http://localhost:3000
2. Build and test the Docker image
docker build -t brain-tasks-app:latest .
docker run -d -p 3000:3000 --name brain-tasks-app-test brain-tasks-app:latest
3. Push the image to DockerHub
docker login
docker tag brain-tasks-app:latest <your-dockerhub-username>/brain-tasks-app:latest
docker push <your-dockerhub-username>/brain-tasks-app:latest
4. Configure Jenkins
	1. Install Jenkins on an EC2 instance (or your preferred host) and visit http://<ec2-public-ip>:8080 to unlock it.
	2. Install plugins: Docker Pipeline, Git, Kubernetes CLI, Pipeline, Pipeline: AWS Steps.
	3. Add credentials under Manage Jenkins → Credentials: 
		○ dockerhub-creds — DockerHub username/token
		○ aws-creds — AWS access key/secret (or attach an IAM role to the Jenkins host)
	4. Add a GitHub webhook pointing to http://<ec2-public-ip>:8080/github-webhook/.
5. Create the EKS cluster
eksctl create cluster --name brain-tasks-cluster --region us-east-1 \
  --nodegroup-name brain-tasks-nodes --node-type t3.medium \
  --nodes 2 --nodes-min 1 --nodes-max 3 --managed
aws eks --region us-east-1 update-kubeconfig --name brain-tasks-cluster
kubectl get nodes
6. Deploy to Kubernetes
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl get svc brain-tasks-app-service   # note the EXTERNAL-IP
7. Set up the Jenkins pipeline
Create a Pipeline job in Jenkins pointing at this repo's Jenkinsfile, with the GitHub webhook trigger enabled.
From this point on, every git push to main automatically builds, pushes, and deploys the app.
8. Monitoring
aws eks update-cluster-config \
  --name brain-tasks-cluster \
  --region us-east-1 \
  --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}'
kubectl logs -l app=brain-tasks-app --tail=100 -f
Build and deploy logs are available in Jenkins → job → build number → Console Output; cluster and control-plane logs stream to CloudWatch Logs.

🔄 Pipeline Explanation
The Jenkinsfile defines a declarative pipeline with four stages:
Stage	Description
1. Checkout	Pulls the latest code from GitHub
2. Build Docker Image	Builds the app into a Docker image
3. Push to DockerHub	Logs in and pushes the image with the build-number and latest tags
4. Deploy to Kubernetes	Applies the manifests in k8s/ and rolls out the new image on the EKS cluster
The pipeline is triggered automatically by a GitHub webhook on every push to main.
pipeline {
    agent any
environment {
        DOCKERHUB_CREDS = credentials('dockerhub-creds')
        IMAGE_NAME       = "<your-dockerhub-username>/brain-tasks-app"
        IMAGE_TAG        = "${env.BUILD_NUMBER}"
        AWS_REGION       = "us-east-1"
        CLUSTER_NAME     = "brain-tasks-cluster"
    }
stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/Vennilavanguvi/Brain-Tasks-App.git'
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

🌐 Live Deployment
Item	Value
Application URL	http://<LoadBalancer external IP or DNS>
LoadBalancer ARN	<paste ARN here>

📸 Screenshot Evidence
Each stage below should be documented with a screenshot proving it worked end-to-end.
1. App Running Locally
Browser at localhost:3000 showing the React app loaded, alongside the terminal running npm start.
<screenshot>
2. Docker Build Success
Terminal showing docker build -t brain-tasks-app:latest . completing without errors.
<screenshot>
3. Docker Container Running
Terminal output of docker ps showing the brain-tasks-app-test container as Up, and the browser at localhost:3000 showing the app served from the container.
<screenshot>
4. DockerHub Repository
DockerHub repo page showing the pushed image tags.
<screenshot>
5. GitHub Repository
The repo's main page on GitHub showing all project files.
<screenshot>
6. GitHub Webhook Configured
GitHub repo → Settings → Webhooks page showing the webhook URL and a green checkmark for a successful recent delivery.
<screenshot>
7. Jenkins Dashboard
Jenkins login/unlock screen and dashboard, proving the initial setup was completed.
<screenshot>
8. EKS Cluster Running
Terminal output of kubectl get nodes showing node(s) in Ready status, and the AWS Console → EKS page showing the cluster status as Active.
<screenshot>
9. Kubernetes Deployment Applied
Terminal output of kubectl get pods showing pods Running, and kubectl get svc brain-tasks-app-service showing TYPE: LoadBalancer with an EXTERNAL-IP.
<screenshot>
10. Jenkins Pipeline Build
Jenkins pipeline job page showing a successful build in history, the stage view (Checkout → Build → Push → Deploy, all green), and the console output of a build.
<screenshot>
11. Auto-Trigger Proof (Real CI/CD)
A git push in the terminal, immediately followed by Jenkins showing a new build auto-started.
<screenshot>
12. App Live on the Internet
Browser showing the app loaded from the LoadBalancer's external URL.
<screenshot>
13. LoadBalancer ARN
AWS Console → EC2 → Load Balancers, showing the load balancer with its ARN visible.
<screenshot>
14. CloudWatch Logs
CloudWatch Logs console showing EKS cluster log groups with recent entries.
<screenshot>

📄 License
This project is provided for educational/assignment purposes.

From <https://claude.ai/chat/81c99288-96c8-4c88-bccb-1880b21bf27c> 
