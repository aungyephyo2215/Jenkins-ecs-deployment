pipeline {
  agent any

  environment {
    AWS_DEFAULT_REGION     = 'ap-southeast-1'
    LOG_FILE               = 's3list.txt'
    AWS_ECS_CLUSTER        = 'jenkin-learn-app-test-env'
    AWS_ECS_SERVICE_PROD   = 'learn-app-service'
    AWS_ECS_TD_PROD        = 'Jenkins-learn-app'
  }

  stages {

    stage('Build App') {
      agent {
        docker {
          image 'node:18-bullseye'
          args '--user=root'  // ✅ Debian-based Node
          reuseNode true
        }
      }
      steps {
        deleteDir() // ✅ Clean workspace
        sh '''
          node --version
          npm --version
          npm ci
          npm run build
        '''
        stash name: 'build', includes: 'build/**'
        stash name: 'node_modules', includes: 'node_modules/**'
      }
    }

    stage('Build Docker Image') {
      agent {
        label 'docker-host' // ✅ Use Jenkins node with Docker installed
      }
      steps {
        unstash 'build'
        sh '''
          docker build -t myjenkinsapp .
        '''
      }
    }

    stage('Register & Deploy ECS Task') {
      agent {
        docker {
          image 'amazonlinux:2'
          args '--user=root' // ✅ safer than entrypoint override
          reuseNode true
        }
      }
      steps {
        withCredentials([
          usernamePassword(
            credentialsId: 'my-aws',
            usernameVariable: 'AWS_ACCESS_KEY_ID',
            passwordVariable: 'AWS_SECRET_ACCESS_KEY'
          )
        ]) {
          sh """
            set -e

            echo '=== Installing CLI Tools ===' > \$LOG_FILE
            yum install -y jq aws-cli >> \$LOG_FILE 2>&1

            echo '=== AWS CLI Version ===' >> \$LOG_FILE
            aws --version >> \$LOG_FILE 2>&1

            echo '\\n=== Register ECS Task Definition ===' >> \$LOG_FILE
            LATEST_TD_REVISION=\$(aws ecs register-task-definition \\
              --cli-input-json file://aws/task-definition-prod.json | jq -r '.taskDefinition.revision')

            echo "Revision: \$LATEST_TD_REVISION" >> \$LOG_FILE

            echo '\\n=== Update ECS Service ===' >> \$LOG_FILE
            aws ecs update-service \\
              --cluster ${AWS_ECS_CLUSTER} \\
              --service ${AWS_ECS_SERVICE_PROD} \\
              --task-definition ${AWS_ECS_TD_PROD}:\$LATEST_TD_REVISION >> \$LOG_FILE 2>&1

            aws ecs wait services-stable \\
              --cluster ${AWS_ECS_CLUSTER} \\
              --services ${AWS_ECS_SERVICE_PROD}
          """
        }
      }
      post {
        always {
          archiveArtifacts artifacts: "${LOG_FILE}", fingerprint: true
        }
      }
    }
  }

  post {
    always {
      cleanWs() // ✅ Clean up after build
    }
  }
}
