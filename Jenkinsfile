pipeline {
  agent any

  environment {
    NPM_CONFIG_CACHE = '/tmp/.npm-cache'
  }

  stages {
    stage('Build') {
      agent {
        docker {
          image 'node:18-alpine'
          reuseNode true
        }
      }
      steps {
        sh '''
          ls -la
          node --version
          npm --version
          npm ci
          npm run build
          ls -la
        '''
        stash name: 'build', includes: 'build/**'
        stash name: 'node_modules', includes: 'node_modules/**'
      }
    }

    stage('Tests') {
      parallel {
        stage('Unit tests') {
          agent {
            docker {
              image 'node:18-alpine'
              reuseNode true
            }
          }
          steps {
            unstash 'node_modules'
            sh '''
              npm test
            '''
          }
          post {
            always {
              junit 'test-results/junit.xml'
            }
          }
        }

        stage('E2E') {
          agent {
            docker {
              image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
              reuseNode true
            }
          }
          steps {
            unstash 'build'
            sh '''
              npm install serve
              npx serve -s build &
              sleep 10
              npx playwright test --reporter=html
            '''
          }
          post {
            always {
              publishHTML([
                allowMissing: false,
                alwaysLinkToLastBuild: false,
                icon: '',
                keepAll: false,
                reportDir: 'playwright-report',
                reportFiles: 'index.html',
                reportName: 'Playwright HTML Report',
                useWrapperFileDirectly: true
              ])
            }
          }
        }
      }
    }

    stage('Deploy') {
      agent {
        docker {
          image 'node:18-bullseye'
          args '-u root:root' // ✅ Run as root to avoid apt/npm issues
          reuseNode true
        }
      }
      environment {
        NPM_CONFIG_CACHE = '/tmp/.npm-cache'
        NETLIFY_AUTH_TOKEN = credentials('netlify-auth-token')
        NETLIFY_SITE_ID = credentials('netlify-site-id')
      }
      steps {
        unstash 'node_modules'
        unstash 'build'
        sh '''
          apt-get update && apt-get install -y git python3 make g++
          npm install -g netlify-cli@20.1.1
          echo "Deploying to Netlify with site ID: $NETLIFY_SITE_ID"
          netlify --version
          netlify status 
          netlify deploy --prod --dir=build --auth=$NETLIFY_AUTH_TOKEN --site=$NETLIFY_SITE_ID  
        '''
      }
    }
    stage('E2E-Prod') {
      agent {
        docker {
          image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
          reuseNode true
        }
      }
      environment {
        CI_ENVIRONMENT_URL = 'https://sunny-tartufo-84b220.netlify.app'
          }
      steps {
        unstash 'build'
        sh '''
          npx playwright test --reporter=html 
          
        '''
      }
      post {
        always {
          publishHTML([
            allowMissing: false,
            alwaysLinkToLastBuild: false,
            icon: '',
            keepAll: false,
            reportDir: 'playwright-report',
            reportFiles: 'index.html',
            reportName: 'Playwright HTML Report (E2E-Prod)',
            useWrapperFileDirectly: true
          ])
        }
      }
    }
  }
}
