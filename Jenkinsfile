pipeline {
  agent any
  environment {
    AWS_REGION = 'ap-south-1'
    REGISTRY   = '413816840602.dkr.ecr.ap-south-1.amazonaws.com'
  }
  stages {
    stage('Checkout') { steps { checkout scm } }
    stage('ECR login') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'aws-ecr',
          usernameVariable: 'AWS_ACCESS_KEY_ID',
          passwordVariable: 'AWS_SECRET_ACCESS_KEY'
        )]) {
          sh '''
            aws ecr get-login-password --region "$AWS_REGION" \
              | docker login --username AWS --password-stdin "$REGISTRY"
          '''
        }
      }
    }
    stage('proxy') {
      steps {
        sh '''
          docker build -t $REGISTRY/week6-proxy:$GIT_COMMIT -t $REGISTRY/week6-proxy:latest ./proxy
          docker push $REGISTRY/week6-proxy:$GIT_COMMIT
          docker push $REGISTRY/week6-proxy:latest
        '''
      }
    }
    stage('backend') {
      steps {
        sh '''
          docker build -t $REGISTRY/week6-backend:$GIT_COMMIT -t $REGISTRY/week6-backend:latest ./backend
          docker push $REGISTRY/week6-backend:$GIT_COMMIT
          docker push $REGISTRY/week6-backend:latest
        '''
      }
    }
  }
}
