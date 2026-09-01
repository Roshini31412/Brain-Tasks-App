pipeline {
  agent any
  environment {
    DOCKERHUB_CREDS = credentials('dockerhub-creds')
    DOCKERHUB_USER  = "roshini31"
    IMAGE_NAME      = "brain-tasks-app"
    IMAGE_TAG       = "${env.BUILD_NUMBER}"
    AWS_REGION      = "ap-south-1"
    CLUSTER_NAME    = "brain-tasks-cluster"
  }
  stages {
    stage('Checkout') {
      steps { checkout scm }
    }
    stage('Docker Build') {
      steps {
        sh "docker build -t ${DOCKERHUB_USER}/${IMAGE_NAME}:${IMAGE_TAG} ."
        sh "docker tag ${DOCKERHUB_USER}/${IMAGE_NAME}:${IMAGE_TAG} ${DOCKERHUB_USER}/${IMAGE_NAME}:latest"
      }
    }
    stage('Docker Push') {
      steps {
        sh "echo $DOCKERHUB_CREDS_PSW | docker login -u $DOCKERHUB_CREDS_USR --password-stdin"
        sh "docker push ${DOCKERHUB_USER}/${IMAGE_NAME}:${IMAGE_TAG}"
        sh "docker push ${DOCKERHUB_USER}/${IMAGE_NAME}:latest"
      }
    }
    stage('Configure kubeconfig') {
      steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
          sh "aws eks update-kubeconfig --name ${CLUSTER_NAME} --region ${AWS_REGION}"
        }
      }
    }
    stage('Deploy to EKS') {
      steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
          sh "kubectl set image deployment/${IMAGE_NAME} ${IMAGE_NAME}=${DOCKERHUB_USER}/${IMAGE_NAME}:${IMAGE_TAG}"
          sh "kubectl rollout status deployment/${IMAGE_NAME} --timeout=120s"
        }
      }
    }
    stage('Verify') {
      steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
          sh "kubectl get pods -o wide"
          sh "kubectl get svc ${IMAGE_NAME}-service"
        }
      }
    }
  }
  post {
    success { echo 'Pipeline succeeded — app deployed to EKS.' }
    failure { echo 'Pipeline failed — check stage logs above.' }
    always  { sh 'docker logout' }
  }
}
