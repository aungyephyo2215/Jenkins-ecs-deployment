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

              aws ecs register-task-definition --cli-input-json file://aws/task-definition-prod.json
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
