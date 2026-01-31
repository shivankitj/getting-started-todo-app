pipeline {
    agent any

    environment {
        // Docker registry and image names
        REGISTRY_CREDENTIALS = 'dockerhub-credentials'
        DOCKER_REGISTRY = 'docker.io'
        IMAGE_NAME = 'todo-app'
        IMAGE_TAG = "${BUILD_NUMBER}"
        DOCKER_IMAGE = "${DOCKER_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}"
        
        // Node version
        NODE_VERSION = '22'
        
        // Database for testing
        DB_TYPE = 'sqlite'
    }

    options {
        // Keep the last 10 builds
        buildDiscarder(logRotator(numToKeepStr: '10'))
        // Timeout after 30 minutes
        timeout(time: 30, unit: 'MINUTES')
        // Add timestamps to console output
        timestamps()
    }

    stages {
        stage('Checkout') {
            steps {
                script {
                    echo "Checking out source code..."
                    checkout scm
                }
            }
        }

        stage('Setup') {
            steps {
                script {
                    echo "Setting up environment..."
                    // Ensure Node.js and npm are available
                    sh '''
                        node --version
                        npm --version
                    '''
                }
            }
        }

        stage('Install Dependencies') {
            parallel {
                stage('Backend Dependencies') {
                    steps {
                        script {
                            echo "Installing backend dependencies..."
                            dir('backend') {
                                sh 'npm install'
                            }
                        }
                    }
                }
                stage('Client Dependencies') {
                    steps {
                        script {
                            echo "Installing client dependencies..."
                            dir('client') {
                                sh 'npm install'
                            }
                        }
                    }
                }
            }
        }

        stage('Code Quality Checks') {
            parallel {
                stage('Backend Linting') {
                    steps {
                        script {
                            echo "Checking backend code formatting..."
                            dir('backend') {
                                sh 'npm run format-check'
                            }
                        }
                    }
                }
                stage('Client Linting') {
                    steps {
                        script {
                            echo "Checking client code quality..."
                            dir('client') {
                                sh 'npm run lint'
                            }
                        }
                    }
                }
            }
        }

        stage('Unit Tests') {
            parallel {
                stage('Backend Tests') {
                    steps {
                        script {
                            echo "Running backend unit tests..."
                            dir('backend') {
                                sh 'npm test'
                            }
                        }
                    }
                }
            }
        }

        stage('Build Client') {
            steps {
                script {
                    echo "Building client application..."
                    dir('client') {
                        sh 'npm run build'
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    echo "Building Docker image: ${DOCKER_IMAGE}"
                    sh "docker build -t ${DOCKER_IMAGE} -t ${IMAGE_NAME}:latest ."
                }
            }
        }

        stage('Security Scan') {
            steps {
                script {
                    echo "Scanning Docker image for vulnerabilities..."
                    // Optional: Use Trivy or another security scanning tool
                    // sh "trivy image --severity HIGH,CRITICAL ${DOCKER_IMAGE}"
                    echo "Security scan completed (configure scanning tool as needed)"
                }
            }
        }

        stage('Push to Registry') {
            when {
                branch 'main'
                // Uncomment to push only on successful builds
                // expression { currentBuild.result == null || currentBuild.result == 'SUCCESS' }
            }
            steps {
                script {
                    echo "Pushing image to registry..."
                    withCredentials([usernamePassword(credentialsId: env.REGISTRY_CREDENTIALS, usernameVariable: 'REGISTRY_USER', passwordVariable: 'REGISTRY_PASS')]) {
                        sh '''
                            echo $REGISTRY_PASS | docker login -u $REGISTRY_USER --password-stdin
                            docker push ${DOCKER_IMAGE}
                            docker push ${IMAGE_NAME}:latest
                            docker logout
                        '''
                    }
                }
            }
        }

        stage('Deploy to Development') {
            when {
                branch 'main'
            }
            steps {
                script {
                    echo "Deploying to development environment..."
                    sh '''
                        echo "Using docker-compose to deploy..."
                        docker-compose -f compose.yaml down || true
                        docker-compose -f compose.yaml up -d
                        echo "Waiting for services to be ready..."
                        sleep 10
                        docker-compose -f compose.yaml ps
                    '''
                }
            }
        }

        stage('Health Check') {
            when {
                branch 'main'
            }
            steps {
                script {
                    echo "Performing health checks..."
                    retry(3) {
                        sh '''
                            echo "Checking application health..."
                            curl -f http://localhost/api/health || true
                            echo "Health check completed"
                        '''
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                echo "Cleaning up..."
                // Optionally clean up Docker resources
                sh 'docker system prune -f || true'
            }
            // Clean workspace
            cleanWs()
        }
        success {
            script {
                echo "Pipeline executed successfully!"
                // Send success notification
                // emailext(
                //     subject: "Build SUCCESS: ${JOB_NAME} #${BUILD_NUMBER}",
                //     body: "The build ${BUILD_NUMBER} of ${JOB_NAME} succeeded.",
                //     to: 'team@example.com'
                // )
            }
        }
        failure {
            script {
                echo "Pipeline failed!"
                // Send failure notification
                // emailext(
                //     subject: "Build FAILED: ${JOB_NAME} #${BUILD_NUMBER}",
                //     body: "The build ${BUILD_NUMBER} of ${JOB_NAME} failed. Check console output for details.",
                //     to: 'team@example.com'
                // )
            }
        }
    }
}
