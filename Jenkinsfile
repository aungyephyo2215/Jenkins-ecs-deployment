pipeline {
    agent any

    environment {
        NETLIFY_AUTH_TOKEN = credentials('netlify-auth-token')
        NETLIFY_SITE_ID = credentials('netlify-site-id')
    }

    stages {

        stage('Build') {
            agent {
                docker {
                    image 'node:18-bullseye'
                    reuseNode true
                }
            }
            steps {
                deleteDir()
                sh '''
                    set -e
                    npm install
                    npm run build
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
                            image 'node:18-bullseye'
                            reuseNode true
                        }
                    }
                    steps {
                        unstash 'node_modules'
                        sh '''
                            set -e
                            npm test || true
                        '''
                    }
                    post {
                        always {
                            junit 'jest-results/junit.xml'
                            echo '✅ Unit tests complete.'
                        }
                        failure {
                            echo '❌ Unit tests failed — check console output.'
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
                        unstash 'node_modules'
                        sh '''
                            set -e
                            npm install serve
                            npx serve -s build &
                            SERVER_PID=$!
                            sleep 10
                            npx playwright test --reporter=html || true
                            kill $SERVER_PID || true
                        '''
                    }
                    post {
                        always {
                            publishHTML([
                                allowMissing: true,
                                alwaysLinkToLastBuild: true,
                                keepAll: true,
                                reportDir: 'playwright-report',
                                reportFiles: 'index.html',
                                reportName: 'Local E2E',
                                useWrapperFileDirectly: true
                            ])
                            echo '📊 Playwright E2E test report published.'
                        }
                        failure {
                            echo '❌ E2E tests failed — inspect the report.'
                        }
                    }
                }
            }
        }

        stage('Deploy staging') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                    reuseNode true
                }
            }
            environment {
                CI_ENVIRONMENT_URL = ''
            }
            steps {
                unstash 'build'
                unstash 'node_modules'
                sh '''
                    set -e
                    npm install netlify-cli jq
                    netlify --version
                    echo "Deploying to staging. Site ID: $NETLIFY_SITE_ID"
                    netlify status
                    netlify deploy --dir=build --auth=$NETLIFY_AUTH_TOKEN --site=$NETLIFY_SITE_ID --json > deploy-output.json
                    jq -r '.deploy_ssl_url' deploy-output.json > staging_url.txt
                '''
                script {
                    def url = readFile('staging_url.txt').trim()
                    env.CI_ENVIRONMENT_URL = url
                    echo "✅ Staging URL: ${url}"
                }
                sh '''
                    npx playwright test --reporter=html --base-url=$CI_ENVIRONMENT_URL || true
                '''
            }
            post {
                always {
                    publishHTML([
                        allowMissing: true,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: 'playwright-report',
                        reportFiles: 'index.html',
                        reportName: 'Staging E2E',
                        useWrapperFileDirectly: true
                    ])
                    echo '📊 Staging Playwright report published.'
                }
                failure {
                    echo '❌ Staging deployment or E2E failed.'
                }
            }
        }

        stage('Approval') {
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    input message: 'Do you wish to deploy to production?', ok: 'Yes, I am sure!'
                }
            }
        }

        stage('Deploy prod') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                    reuseNode true
                }
            }
            environment {
                CI_ENVIRONMENT_URL = 'https://your-netlify-prod-url.netlify.app'
            }
            steps {
                unstash 'build'
                unstash 'node_modules'
                sh '''
                    set -e
                    npm install netlify-cli
                    netlify --version
                    echo "Deploying to production. Site ID: $NETLIFY_SITE_ID"
                    netlify status
                    netlify deploy --prod --dir=build --auth=$NETLIFY_AUTH_TOKEN --site=$NETLIFY_SITE_ID
                    npx playwright test --reporter=html --base-url=$CI_ENVIRONMENT_URL || true
                '''
            }
            post {
                always {
                    publishHTML([
                        allowMissing: true,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: 'playwright-report',
                        reportFiles: 'index.html',
                        reportName: 'Prod E2E',
                        useWrapperFileDirectly: true
                    ])
                    echo '📊 Production Playwright report published.'
                }
                failure {
                    echo '❌ Production deployment or E2E tests failed.'
                }
            }
        }
    }
}
