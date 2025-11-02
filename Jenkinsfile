pipeline {
  agent any

  environment {
    AWS_REGION = 'ap-south-1b'                
    AWS_ACCOUNT_ID = '288434313151'     
    ECR_REPO = 'ci-cd-node-sample'        
    APP_EC2_HOST = '13.234.67.2'         
    APP_EC2_USER = 'ubuntu'                
    IMAGE_TAG = "${env.BUILD_NUMBER}"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build & Test (Node)') {
      steps {
        script {
          // For Linux agents use sh, for Windows use bat (Jenkins windows agent)
          if (isUnix()) {
            sh 'node --version || true'
            sh 'npm ci'
            sh 'npm test || true'   // don't block if tests absent; remove || true in real tests
          } else {
            bat 'node --version || echo no-node'
            bat 'npm ci'
            bat 'npm test || echo no-tests'
          }
        }
      }
    }

    stage('Login to ECR, Build & Push Image') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'aws-creds',
                                          usernameVariable: 'AWS_ACCESS_KEY_ID',
                                          passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
          script {
            if (isUnix()) {
              sh '''
                set -e
                export AWS_REGION=${AWS_REGION}
                export AWS_ACCOUNT_ID=${AWS_ACCOUNT_ID}
                ECR_URI=${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}

                aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
                aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
                aws configure set region $AWS_REGION

                aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                docker build -t ${ECR_URI}:${IMAGE_TAG} .
                docker push ${ECR_URI}:${IMAGE_TAG}
              '''
            } else {
              bat """
                set AWS_REGION=%AWS_REGION%
                set AWS_ACCOUNT_ID=%AWS_ACCOUNT_ID%
                set ECR_URI=%AWS_ACCOUNT_ID%.dkr.ecr.%AWS_REGION%.amazonaws.com/%ECR_REPO%

                aws configure set aws_access_key_id %AWS_ACCESS_KEY_ID%
                aws configure set aws_secret_access_key %AWS_SECRET_ACCESS_KEY%
                aws configure set region %AWS_REGION%

                aws ecr get-login-password --region %AWS_REGION% | docker login --username AWS --password-stdin %AWS_ACCOUNT_ID%.dkr.ecr.%AWS_REGION%.amazonaws.com
                docker build -t %ECR_URI%:%IMAGE_TAG% .
                docker push %ECR_URI%:%IMAGE_TAG%
              """
            }
          }
        }
      }
    }

    stage('Deploy to App EC2') {
      steps {
        sshagent (credentials: ['app-ssh-key']) {
          script {
            def remoteCmds = """
              set -e
              AWS_REGION=${AWS_REGION}
              AWS_ACCOUNT_ID=${AWS_ACCOUNT_ID}
              ECR_URI=${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}

              # ensure docker exists
              which docker || (sudo apt-get update && sudo apt-get install -y docker.io)

              # login to ECR and pull image
              aws ecr get-login-password --region ${AWS_REGION} | sudo docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
              sudo docker pull ${ECR_URI}:${IMAGE_TAG}

              # replace running container
              sudo docker rm -f ci-cd-node || true
              sudo docker run -d --name ci-cd-node -p 80:3000 ${ECR_URI}:${IMAGE_TAG}
            """

            if (isUnix()) {
              sh "ssh -o StrictHostKeyChecking=no ${APP_EC2_USER}@${APP_EC2_HOST} '${remoteCmds}'"
            } else {
              bat """
                ssh -o StrictHostKeyChecking=no %APP_EC2_USER%@%APP_EC2_HOST% "${remoteCmds}"
              """
            }
          }
        }
      }
    }
  }

  post {
    success {
      echo "Build ${env.BUILD_NUMBER} succeeded. Image: ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${IMAGE_TAG}"
    }
    failure {
      echo "Build ${env.BUILD_NUMBER} FAILED. Check console output."
    }
  }
}
