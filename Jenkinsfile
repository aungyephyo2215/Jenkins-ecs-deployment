pipeline {
  agent any

  environment {
    AWS_DEFAULT_REGION = 'ap-southeast-1'
    LOG_FILE = 's3list.txt'
  }

  stages {
    stage('Check AWS CLI') {
      agent {
        docker {
          image 'amazon/aws-cli'
          args '--entrypoint=""'
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
          script {
            sh """
              yum install jq -y
              echo '=== AWS CLI Version ===' > \$LOG_FILE 2>&1
              aws --version >> \$LOG_FILE 2>&1
              echo '\\n=== Register ECS Task Definition ===' >> \$LOG_FILE 2>&1
              $LATEST_TD_REVISION = aws ecs register-task-definition --cli-input-json file://aws/task-definition-prod.json | jq '.taskDefinition.revison' >> \$LOG_FILE 2>&1
              echo $LATEST_TD_REVISION
              aws ecs update-service --cluster Jenkins-lab-wordy-hippopotamus-pwnh7o --service Jenkins-learn-app-service-gc86u70w --task-definition Jenkins-learn-app:$LATEST_TD_REVISION >> \$LOG_FILE 2>&1

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

  // ✅ Clean workspace after entire pipeline
  post {
    always {
      cleanWs()
    }
  }
}
