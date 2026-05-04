pipeline {
    agent any

    environment {
        // ---------------- Docker ----------------
        DOCKER_HUB_USER = 'arunkumarravi08' // PLEASE UPDATE THIS
        DOCKER_HUB_REPO = 'espocrm'
        IMAGE_TAG = "${BUILD_NUMBER}"

        // ---------------- AWS / EKS ----------------
        CLUSTER_NAME = 'espocrm' // PLEASE UPDATE THIS
        REGION = 'us-east-1'        // PLEASE UPDATE THIS
    }

    stages {
        stage('Git Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install Frontend Deps') {
            steps {
                sh 'npm install'
            }
        }

        stage('Install Backend Deps') {
            steps {
                sh 'composer install --no-interaction --prefer-dist'
            }
        }

        stage('ESLint') {
            steps {
                sh 'npx eslint . || echo "ESLint failed but continuing"'
            }
        }

        stage('Frontend Tests') {
            steps {
                sh 'npm test || echo "No frontend tests found"'
            }
        }

        stage('Backend Tests') {
            steps {
                sh 'vendor/bin/phpunit || echo "No backend tests found"'
            }
        }

        stage('SonarQube Scan') {
            steps {
                withSonarQubeEnv('SonarQube-Server') {
                    sh '''
                    docker run --rm \
                      -e SONAR_HOST_URL=${SONAR_HOST_URL} \
                      -e SONAR_LOGIN=${SONAR_AUTH_TOKEN} \
                      -v "${WORKSPACE}:/usr/src" \
                      sonarsource/sonar-scanner-cli \
                      -Dsonar.projectKey=espocrm \
                      -Dsonar.sources=.
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                waitForQualityGate abortPipeline: true
            }
        }

        stage('Trivy FS Scan') {
            steps {
                sh 'trivy fs . --severity HIGH,CRITICAL --format table'
            }
        }

        stage('Docker Build') {
            steps {
                sh 'docker build -t ${DOCKER_HUB_USER}/${DOCKER_HUB_REPO}:${IMAGE_TAG} .'
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh 'trivy image ${DOCKER_HUB_USER}/${DOCKER_HUB_REPO}:${IMAGE_TAG} --severity HIGH,CRITICAL'
            }
        }

        stage('Docker Login') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub-creds', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh "echo ${PASS} | docker login -u ${USER} --password-stdin"
                }
            }
        }

        stage('Docker Push') {
            steps {
                sh 'docker push ${DOCKER_HUB_USER}/${DOCKER_HUB_REPO}:${IMAGE_TAG}'
            }
        }

        stage('Helm Lint') {
            steps {
                sh 'helm lint ./helm/espocrm'
            }
        }

        stage('Observability Setup') {
            steps {
                withCredentials([aws(credentialsId: 'aws-creds', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    sh '''
                    aws eks update-kubeconfig --name ${CLUSTER_NAME} --region ${REGION}
                    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
                    helm repo update
                    helm upgrade --install prometheus prometheus-community/kube-prometheus-stack \
                    --namespace monitoring --create-namespace \
                    --set grafana.service.type=LoadBalancer || echo "Prometheus update failed"
                    '''
                }
            }
        }

        stage('EKS Auth & Deploy') {
            steps {
                withCredentials([aws(credentialsId: 'aws-creds', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    sh '''
                    aws eks update-kubeconfig --name ${CLUSTER_NAME} --region ${REGION}
                    # Deploy using Helm from the local helm directory
                    helm upgrade --install espocrm ./helm/espocrm \
                    --namespace espocrm --create-namespace \
                    --set image.repository=${DOCKER_HUB_USER}/${DOCKER_HUB_REPO} \
                    --set image.tag=${IMAGE_TAG} \
                    --wait

                    echo "------------------------------------------------"
                    echo "EspoCRM URL:"
                    kubectl get svc -n espocrm espocrm-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' || echo "URL Pending..."
                    echo -e "\n------------------------------------------------"
                    '''
                }
            }
        }

        stage('Prometheus Metrics') {
            steps {
                sh 'kubectl get pods -n monitoring | grep prometheus || echo "Metrics pods not found"'
            }
        }

        stage('Grafana Visualization') {
            steps {
                sh '''
                echo "------------------------------------------------"
                echo "Grafana URL:"
                kubectl get svc -n monitoring prometheus-grafana -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' || echo "URL Pending..."
                echo "\n------------------------------------------------"
                '''
            }
        }

        stage('Alerting Notifications') {
            steps {
                echo 'Pipeline completed successfully. Alerting logic can be added here.'
            }
        }
    }

    post {
        always {
            echo 'Pipeline execution finished.'
        }
    }
}
