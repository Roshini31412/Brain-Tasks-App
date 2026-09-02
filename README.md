🚀 Brain-Tasks-App — React App CI/CD Deployment on AWS EKS
<p align="center"> <img src="https://img.shields.io/badge/React-18-blue?logo=react" alt="React" /> <img src="https://img.shields.io/badge/Docker-Container-2496ED?logo=docker&logoColor=white" alt="Docker" /> <img src="https://img.shields.io/badge/DockerHub-Registry-2496ED?logo=docker&logoColor=white" alt="DockerHub" /> <img src="https://img.shields.io/badge/Jenkins-CI%2FCD-D24939?logo=jenkins&logoColor=white" alt="Jenkins" /> <img src="https://img.shields.io/badge/Kubernetes-AWS%20EKS-326CE5?logo=kubernetes&logoColor=white" alt="Kubernetes" /> <img src="https://img.shields.io/badge/Monitoring-CloudWatch-FF9900?logo=amazonaws&logoColor=white" alt="Monitoring" /> </p> <p align="center"> Taking a React application from source code to a fully automated, production-ready deployment — using Docker, Jenkins, DockerHub, and AWS EKS. </p> 

📑 **Table of Contents**
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


<img width="873" height="105" alt="image" src="https://github.com/user-attachments/assets/e2052581-9a67-44bd-819b-903f40f014ac" />


Every push to main automatically triggers Jenkins to build a new Docker image, push it to DockerHub, and roll it out to the Kubernetes cluster — no manual steps required after the initial setup.

**🛠 Tech Stack**

Layer	Tool
Application	React
Containerization	Docker
CI/CD	Jenkins
Image Registry	DockerHub
Orchestration	Kubernetes on AWS EKS
Monitoring	AWS CloudWatch

📁 Project Structure
<img width="605" height="217" alt="image" src="https://github.com/user-attachments/assets/1f53dc3c-e3c3-465a-815f-4dd147c6a14d" />



✅ Prerequisites
	• GitHub, AWS, and DockerHub accounts
	• Installed locally: 
		○ git
		○ node (v18+)
		○ docker
		○ aws-cli
		○ kubectl
		○ eksctl

**⚙️ Setup Instructions**

**1. **Clone and run locally****
git clone https://github.com/Username/Brain-Tasks-App.git
cd Brain-Tasks-App
npm install
npm start          # runs on http://localhost:3000

**2. **Build and test the Docker image****
docker build -t brain-tasks-app:latest .
docker run -d -p 3000:3000 --name brain-tasks-app-test brain-tasks-app:latest

**3. Push the image to DockerHub**
docker login
docker tag brain-tasks-app:latest <your-dockerhub-username>/brain-tasks-app:latest
docker push <your-dockerhub-username>/brain-tasks-app:latest

**4. Configure Jenkins**
	1. Install Jenkins on an EC2 instance (or your preferred host) and visit http://<ec2-public-ip>:8080 to unlock it.
	2. Install plugins: Docker Pipeline, Git, Kubernetes CLI, Pipeline, Pipeline: AWS Steps.
	3. Add credentials under Manage Jenkins → Credentials: 
		○ dockerhub-creds — DockerHub username/token
		○ aws-creds — AWS access key/secret (or attach an IAM role to the Jenkins host)
	4. Add a GitHub webhook pointing to http://<ec2-public-ip>:8080/github-webhook/.
    
**5. Create the EKS cluster**
eksctl create cluster --name brain-tasks-cluster --region us-east-1 \
  --nodegroup-name brain-tasks-nodes --node-type t3.medium \
  --nodes 2 --nodes-min 1 --nodes-max 3 --managed
aws eks --region us-east-1 update-kubeconfig --name brain-tasks-cluster
kubectl get nodes

**6. Deploy to Kubernetes**
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl get svc brain-tasks-app-service   # note the EXTERNAL-IP

**7. Set up the Jenkins pipeline**
Create a Pipeline job in Jenkins pointing at this repo's Jenkinsfile, with the GitHub webhook trigger enabled.
From this point on, every git push to main automatically builds, pushes, and deploys the app.

**8. Monitoring**
aws eks update-cluster-config \
  --name brain-tasks-cluster \
  --region us-east-1 \
  --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}'
kubectl logs -l app=brain-tasks-app --tail=100 -f
Build and deploy logs are available in Jenkins → job → build number → Console Output; cluster and control-plane logs stream to CloudWatch Logs.

🔄 **Pipeline Explanation**
The Jenkinsfile defines a declarative pipeline with four stages:
Stage	Description
1. Checkout	Pulls the latest code from GitHub
2. Build Docker Image	Builds the app into a Docker image
3. Push to DockerHub	Logs in and pushes the image with the build-number and latest tags
4. Deploy to Kubernetes	Applies the manifests in k8s/ and rolls out the new image on the EKS cluster
The pipeline is triggered automatically by a GitHub webhook on every push to main.

🌐 Live Deployment
Item	Value
LoadBalancer ARN	:(http://a84fd2b0b0d224ff6bade7608c8df2d3-1141925910.ap-south-1.elb.amazonaws.com)

📸 Screenshot Evidence
Each stage below should be documented with a screenshot proving it worked end-to-end.
1. App Running Locally
Browser at localhost:3000 showing the React app loaded, alongside the terminal running npm start.

<img width="837" height="325" alt="image" src="https://github.com/user-attachments/assets/252f044a-dc07-4744-a8fc-134e5f018516" />

<img width="1822" height="615" alt="image" src="https://github.com/user-attachments/assets/87891156-1cc7-4419-9c5f-08e96d856aa4" />


2. Dockerize and Test Image
Show containerization of the app.
<img width="1887" height="127" alt="image" src="https://github.com/user-attachments/assets/6d85d693-22f3-4559-b40d-001f35410c65" />

Screenshot of docker run -p 3000:3000 with the app accessible.
<img width="1892" height="558" alt="image" src="https://github.com/user-attachments/assets/95d8f9ee-a904-4a40-b6d2-5d8330fa501a" />


3. Push Image to Registry
Verify integration with Docker Hub.

Screenshot of docker push output.
<img width="928" height="298" alt="image" src="https://github.com/user-attachments/assets/c3087eb2-cf19-4556-9b00-38454a53dd21" />

Screenshot of the image visible in the Docker Hub repository.
<img width="1915" height="722" alt="image" src="https://github.com/user-attachments/assets/501f733b-e080-4ff7-8411-3eb1ff280214" />

4. Kubernetes Deployment on EKS

Screenshot of kubectl apply -f k8s/.
<img width="1826" height="342" alt="image" src="https://github.com/user-attachments/assets/8c0f8268-26f6-4222-9a73-b66b7012cb84" />

Screenshot of kubectl get pods showing Running status.
<img width="935" height="272" alt="image" src="https://github.com/user-attachments/assets/e9c7817f-f89e-41c8-8ccf-2b4daf53ad48" />

<img width="937" height="210" alt="image" src="https://github.com/user-attachments/assets/61612e72-448b-46e6-872f-98ed98c34f05" />

Screenshot of kubectl get svc with LoadBalancer EXTERNAL-IP (e.g., http://a84fd2b0b0d224ff6bade7608c8df2d3-1141925910.ap-south-1.elb.amazonaws.com).
<img width="1910" height="602" alt="image" src="https://github.com/user-attachments/assets/2dac9c80-6c5d-47d2-a4f4-8a7ffafea11b" />

5. CI/CD Pipeline Setup
Prove automation via Jenkins.

Screenshot of Jenkins pipeline configuration or buildspec.yml.

<img width="925" height="432" alt="image" src="https://github.com/user-attachments/assets/983268ff-be4c-42cf-b714-1dbac60ee699" />


Screenshot of a successful pipeline run (Build → Push → Deploy).
<img width="1870" height="792" alt="image" src="https://github.com/user-attachments/assets/c8222294-9947-4c1e-a3c7-2788ec49f601" />


Screenshot of GitHub webhook trigger.
<img width="1907" height="787" alt="image" src="https://github.com/user-attachments/assets/24bb9a64-d394-4f85-99e5-cf941b31c8cd" />


6. Monitoring and Logs
Show application health and logging integration.

Screenshot of CloudWatch Logs or Jenkins console output.
<img width="932" height="333" alt="image" src="https://github.com/user-attachments/assets/ce619096-2281-4d2a-9f52-2cabb6e9eab2" />

Screenshot of kubectl describe pod or kubectl logs.
<img width="937" height="173" alt="image" src="https://github.com/user-attachments/assets/7264d340-b8b9-45ba-a42c-d74644b6ab6d" />

Screenshot of CloudWatch Log Groups list (integration proof).

<img width="1911" height="785" alt="image" src="https://github.com/user-attachments/assets/f873a855-0eb6-49d4-b456-67bc9ec85f45" />
<img width="1912" height="677" alt="image" src="https://github.com/user-attachments/assets/6062c071-7513-4b5f-80e6-caff2ac02d50" />
<img width="1916" height="666" alt="image" src="https://github.com/user-attachments/assets/59842041-29ba-4f3a-a93e-26d25bb2d10a" />

Screenshot of Application log stream.

<img width="1902" height="552" alt="image" src="https://github.com/user-attachments/assets/486b1264-a6c5-4d8d-b327-b0d3a8135ce7" />


Screenshot of Jenkins console log (local build/deploy proof).

<img width="1847" height="632" alt="image" src="https://github.com/user-attachments/assets/ecc54400-ab76-471e-8d5a-cc03fd3a1841" />

Screenshot of Cluster dashboard 


<img width="1088" height="828" alt="image" src="https://github.com/user-attachments/assets/a3d6e873-8c2e-443e-b8d2-2c82323570af" />

