pipeline {
  agent any

  environment {
    AWS_DEFAULT_REGION = 'ap-southeast-1'
    LOG_FILE = 's3list.txt'
    AWS_ECS_CLUSTER = 'jenkin-learn-app-test-env'
    AWS_ECS_SERVICE_PROD = 'learn-app-service'
    AWS_ECS_TD_PROD =  'Jenkins-learn-app'

  }

  stages {
    stage('Check AWS CLI') {
      agent {
        docker {
          image 'amazon/aws-cli'
          args '-u root --entrypoint=""'
          reuseNode true
        }
      }

      steps {
        withCredentials([
          usernamePassword(
            credentialsId: 'my-aws',
            passwordVariable: 'AWS_SECRET_ACCESS_KEY',
            usernameVariable: 'AWS_ACCESS_KEY_ID'
          )
        ]) {
            sh """
              echo '=== JQ ===' > \$LOG_FILE 2>&1
              yum install jq -y >> \$LOG_FILE 2>&1
              echo '=== AWS CLI Version ===' >> \$LOG_FILE 2>&1
              aws --version >> \$LOG_FILE 2>&1

              echo '\\n=== Register ECS Task Definition ===' >> \$LOG_FILE 2>&1
              LATEST_TD_REVISION=\$(aws ecs register-task-definition --cli-input-json file://aws/task-definition-prod.json | jq -r '.taskDefinition.revision')
              echo \$LATEST_TD_REVISION >> \$LOG_FILE 2>&1

              echo '\\n=== Register ECS Services Deployment ===' >> \$LOG_FILE 2>&1
              aws ecs update-service --cluster $AWS_ECS_CLUSTER --service Jenkins-$AWS_ECS_SERVICE_PROD --task-definition AWS_ECS_TD_PROD:\$LATEST_TD_REVISION >> \$LOG_FILE 2>&1
              aws ecs wait services-stable --cluster $AWS_ECS_CLUSTER  --services $AWS_ECS_SERVICE_PROD
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

  // ✅ Clean workspace after entire pipeline
  post {
    always {
      cleanWs()
    }
  }
}
