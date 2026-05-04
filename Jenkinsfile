pipeline {
    agent any

    environment {
        // SonarQube Server details (managed in Jenkins Credentials/Systems)
        SONAR_SCANNER_HOME = tool 'sonar-scanner'
        SCANNER_IMAGE      = 'sonarsource/sonar-scanner-cli'
        
        // Docker Repository Details
        DOCKER_IMAGE_NAME  = 'espocrm'
        DOCKER_TAG         = "${BUILD_NUMBER}"
        DOCKER_REGISTRY    = "arunkumar123" // Replace with your DockerHub/ECR username
        
        // AWS EKS Details
        AWS_REGION         = 'us-east-1'
        CLUSTER_NAME       = 'espocrm-cluster'
    }

    stages {
        stage('Initialize & Checkout') {
            steps {
                echo 'Checking out source code...'
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                echo 'Installing Frontend (Npm) and Backend (Composer) dependencies...'
                sh '''
                    npm install --no-fund --no-audit
                    composer install --no-interaction --prefer-dist --optimize-autoloader
                '''
            }
        }

        stage('Static Analysis') {
            steps {
                echo 'Running PHPStan for static code analysis...'
                sh 'npm run sa || echo "Static analysis found issues, but continuing..."'
            }
        }

        stage('Unit Tests') {
            steps {
                echo 'Executing Backend Unit Tests...'
                // These tests must pass for the pipeline to continue
                sh 'vendor/bin/phpunit --colors=always'
            }
        }

        stage('SonarQube Scan') {
            steps {
                script {
                    withSonarQubeEnv('SonarQube-Server') {
                        echo 'Running SonarQube scan via Docker container...'
                        sh """
                        docker run --rm \
                            --user \$(id -u):\$(id -g) \
                            -e SONAR_HOST_URL=${SONAR_HOST_URL} \
                            -e SONAR_TOKEN=${SONAR_AUTH_TOKEN} \
                            -v "${WORKSPACE}:/usr/src" \
                            ${SCANNER_IMAGE} \
                            -Dsonar.projectKey=espocrm \
                            -Dsonar.sources=. \
                            -Dsonar.exclusions=**/node_modules/**,**/vendor/**,**/tests/**,helm/** \
                            -Dsonar.working.directory=/usr/src/.scannerwork
                        """
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                echo 'Waiting for SonarQube Quality Gate report...'
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Security Scan (FileSystem)') {
            steps {
                echo 'Scanning filesystem for vulnerabilities with Trivy...'
                sh 'trivy fs --format table -o trivy-fs-report.html .'
            }
        }

        stage('Docker Build & Tag') {
            steps {
                echo 'Building Docker Image...'
                sh "docker build -t ${DOCKER_REGISTRY}/${DOCKER_IMAGE_NAME}:${DOCKER_TAG} ."
                sh "docker tag ${DOCKER_REGISTRY}/${DOCKER_IMAGE_NAME}:${DOCKER_TAG} ${DOCKER_REGISTRY}/${DOCKER_IMAGE_NAME}:latest"
            }
        }

        stage('Security Scan (Image)') {
            steps {
                echo 'Scanning Docker Image for vulnerabilities...'
                sh "trivy image --severity HIGH,CRITICAL ${DOCKER_REGISTRY}/${DOCKER_IMAGE_NAME}:${DOCKER_TAG}"
            }
        }

        stage('Docker Push') {
            steps {
                echo 'Pushing images to registry...'
                withCredentials([usernamePassword(credentialsId: 'docker-hub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh "echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin"
                    sh "docker push ${DOCKER_REGISTRY}/${DOCKER_IMAGE_NAME}:${DOCKER_TAG}"
                    sh "docker push ${DOCKER_REGISTRY}/${DOCKER_IMAGE_NAME}:latest"
                }
            }
        }

        stage('Helm Lint & Package') {
            steps {
                echo 'Linting Helm Charts...'
                sh 'helm lint helm/espocrm'
            }
        }

        stage('EKS Deploy') {
            steps {
                echo 'Deploying to AWS EKS...'
                withAWS(region: "${AWS_REGION}", credentials: 'aws-creds') {
                    sh "aws eks update-kubeconfig --name ${CLUSTER_NAME} --region ${AWS_REGION}"
                    sh """
                    helm upgrade --install espocrm helm/espocrm \
                        --set image.repository=${DOCKER_REGISTRY}/${DOCKER_IMAGE_NAME} \
                        --set image.tag=${DOCKER_TAG} \
                        --namespace default
                    """
                }
            }
        }
    }

    post {
        always {
            echo 'Pipeline execution finished.'
        }
        success {
            echo 'Deployment successful! 🎉'
        }
        failure {
            echo 'Build failed. Please check the logs above for details. ❌'
        }
    }
}
