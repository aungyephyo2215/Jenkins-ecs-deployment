pipeline {
  agent any

  environment {
   /* AWS_ACCESS_KEY_ID     = credentials('aws-access-key')    
    AWS_SECRET_ACCESS_KEY = credentials('aws-secret-key')    */ 
    AWS_DEFAULT_REGION    = 'ap-southeast-1'                 
  }

  stages {

    stage('Check AWS CLI') {
      agent {
        docker {
          image 'amazon/aws-cli'
          
          args "--entrypoint=''"
          reuseNode true
        }
      }
      steps {
      withCredentials([usernamePassword(credentialsId: 'my-aws', passwordVariable: 'AWS_SECRET_ACCESS_KEY', usernameVariable: 'AWS_ACCESS_KEY_ID')]) {
        sh 'aws s3 ls'
}
      }
    }
  }
}
