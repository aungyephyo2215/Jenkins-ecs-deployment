pipeline {
  agent any

  environment {
    NPM_CONFIG_CACHE = '/tmp/.npm-cache'
    NETLIFY_AUTH_TOKEN = credentials('netlify-auth-token')
    NETLIFY_SITE_ID = credentials('netlify-site-id')
    CI_ENVIRONMENT_URL = 'https://sunny-tartufo-84b220.netlify.app'
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
              image 'my-playwright-image'
              reuseNode true
            }
          }
          steps {
            unstash 'build'
            sh '''
              npx serve -s build -l 5000 > /dev/null 2>&1 &
              SERVER_PID=$!
              sleep 10
              npx playwright test --reporter=html
              kill $SERVER_PID
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

        stage('Deploy & E2E-Staging') {
          agent {
            docker {
              image 'node:18-bullseye'
              args '-u root:root'
              reuseNode true
            }
          }
          steps {
            unstash 'node_modules'
            unstash 'build'
            sh '''
              set -e
              apt-get update && apt-get install -y git python3 make g++ jq
              npm install -g netlify-cli@20.1.1
              echo "Deploying to Staging Env with site ID: $NETLIFY_SITE_ID"
              netlify --version
              netlify status 
              netlify deploy --dir=build --auth=$NETLIFY_AUTH_TOKEN --site=$NETLIFY_SITE_ID --json > deploy-output.json
              echo "=== Netlify deploy-output.json ==="
              cat deploy-output.json
              jq -r '.deploy.ssl_url' deploy-output.json > staging_url.txt
              echo "=== staging_url.txt ==="
              cat staging_url.txt
              npx playwright test --reporter=html 
              echo $CI_ENVIRONMENT_URL
            '''
            /*script {
                def stagingUrl = readFile('staging_url.txt').trim()
                if (!stagingUrl || stagingUrl == "null") {
                  error "Staging URL could not be extracted! Check Netlify deploy output."
                }
                echo "Staging URL: ${stagingUrl}"
                sh """
                  npx playwright test --reporter=html --base-url=${stagingUrl}
                """
              }*/
   
          }
          post {
            always {
              publishHTML([
                allowMissing: false,
                alwaysLinkToLastBuild: false,
                icon: '',
                keepAll: false,
                reportDir: 'playwright-report-staginge2e',
                reportFiles: 'index.html',
                reportName: 'Playwright HTML Report (E2E-Staging)',
                useWrapperFileDirectly: true
              ])
            }
          }
        }   
      
        stage('Deploy & E2E-Prod') {
          agent {
            docker {
              image 'node:18-bullseye'
              args '-u root:root'
              reuseNode true
            }
          }
          environment {
            CI_ENVIRONMENT_URL = 'https://sunny-tartufo-84b220.netlify.app'
          }
          steps {
            unstash 'node_modules'
            unstash 'build'
            sh '''
              apt-get update && apt-get install -y git python3 make g++
              npm install -g netlify-cli@20.1.1
              echo "Deploying to Production Env with site ID: $NETLIFY_SITE_ID"
              netlify --version
              netlify status 
              netlify deploy --prod --dir=build --auth=$NETLIFY_AUTH_TOKEN --site=$NETLIFY_SITE_ID
              npx playwright test --reporter=html 
              echo $CI_ENVIRONMENT_URL
            '''
            /*sh """
              npx playwright test --reporter=html --base-url=$CI_ENVIRONMENT_URL
            """*/

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
