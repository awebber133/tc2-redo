pipeline {
  agent any

  environment {
    ECR_REPO  = '238845559349.dkr.ecr.us-east-1.amazonaws.com/app-repository'
    IMAGE_TAG = "${env.BUILD_ID}"
    AWS_REGION = 'us-east-1'
    EKS_CLUSTER = 'eks-cluster'
  }

  stages {

    stage('Checkout') {
      steps {
        git branch: 'main', url: 'https://github.com/awebber133/tc2-redo.git'
      }
    }

    stage('Build Docker Image') {
      steps {
        sh 'docker build -t $ECR_REPO:$IMAGE_TAG ./app'
      }
    }

    stage('Push to ECR') {
      steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
          sh '''
            aws ecr get-login-password --region $AWS_REGION \
              | docker login --username AWS --password-stdin $ECR_REPO

            docker push $ECR_REPO:$IMAGE_TAG
          '''
        }
      }
    }

    stage('Configure kubeconfig') {
      steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
          sh '''
            echo "=== AWS Identity ==="
            aws sts get-caller-identity

            echo "=== Updating kubeconfig ==="
            aws eks update-kubeconfig --region $AWS_REGION --name $EKS_CLUSTER

            echo "=== Testing cluster access ==="
            kubectl get nodes
          '''
        }
      }
    }

    stage('Deploy with Helm') {
      steps {
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
          sh '''
            helm upgrade --install tc2-redo-app ./helm-chart \
              --namespace jenkins-deploy \
              --create-namespace \
              --set image.repository=$ECR_REPO \
              --set image.tag=$IMAGE_TAG
          '''
        }
      }
    }
  }
}
