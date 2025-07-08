pipeline {
  agent any

  environment {
    AWS_ACCESS_KEY_ID     = credentials('aws-access-key')     // ✅ Jenkins credentials ID
    AWS_SECRET_ACCESS_KEY = credentials('aws-secret-key')     // ✅ Jenkins credentials ID
    AWS_DEFAULT_REGION    = 'ap-southeast-1'                  // or us-east-1, etc.
  }

  stages {

    stage('Check AWS CLI') {
      agent {
        docker {
          image 'amazon/aws-cli'
          args '-u root:root' // optional, if needed for write permissions
          reuseNode true
        }
      }
      steps {
        withCredentials([usernamePassword(credentialsId: 'my-aws', passwordVariable: 'AWS_ACCESS_KEY_ID', usernameVariable: 'AWS_SECRET_ACCESS_KEY')]) {
          sh 'aws --version'
      }
        
      }
    }

    stage('List S3 Buckets') {
      agent {
        docker {
          image 'amazon/aws-cli'
          args '-u root:root'
          reuseNode true
        }
      }
      steps {
        sh 'aws s3 ls'
      }
    }
  }
}
