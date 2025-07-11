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
    stage('Register & Deploy ECS Task') {
      agent {
        docker {
          image 'amazonlinux'
          args '-u root --entrypoint=""'
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
          script {
            sh """
              set -e
              echo '=== Installing Dependencies ===' > \$LOG_FILE
              yum install -y jq aws-cli >> \$LOG_FILE 2>&1

              echo '=== AWS CLI Version ===' >> \$LOG_FILE
              aws --version >> \$LOG_FILE 2>&1

              echo '\\n=== Register ECS Task Definition ===' >> \$LOG_FILE
              LATEST_TD_REVISION=\$(aws ecs register-task-definition \\
                --cli-input-json file://aws/task-definition-prod.json | jq -r '.taskDefinition.revision')

              echo "Revision: \$LATEST_TD_REVISION" >> \$LOG_FILE

              echo '\\n=== Update ECS Service to New Task Definition ===' >> \$LOG_FILE
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
      cleanWs()
    }
  }
}
