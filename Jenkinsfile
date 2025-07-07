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
                    environment {
                        NPM_CONFIG_CACHE = '/tmp/.npm-cache'
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
                    environment {
                        NPM_CONFIG_CACHE = '/tmp/.npm-cache'
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
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            environment {
                NPM_CONFIG_CACHE = '/tmp/.npm-cache'
            }
            steps {
                unstash 'node_modules'
                unstash 'build'
                sh '''
                    npm install netlify-cli@20.1.1
                    npx netlify --version
                    # Add deployment command below (adjust as needed)
                    # npx netlify deploy --dir=build --prod --auth=<token> --site=<site-id>
                '''
            }
        }
    }
}
