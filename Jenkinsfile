pipeline {
    agent any

    tools {
        maven 'Maven 3.8.6'
        jdk 'JDK 17'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build and Test Backend Services') {
            parallel {
                stage('Config Server') {
                    steps {
                        dir('config-server') {
                            sh 'mvn clean test'
                        }
                    }
                }
                stage('Service Registry') {
                    steps {
                        dir('service-registry') {
                            sh 'mvn clean test'
                        }
                    }
                }
                stage('API Gateway') {
                    steps {
                        dir('api-gateway') {
                            sh 'mvn clean test'
                        }
                    }
                }
                stage('Microservice Users') {
                    steps {
                        dir('microservice-users') {
                            sh 'mvn clean test'
                        }
                    }
                }
                stage('Microservice Products') {
                    steps {
                        dir('microservice-products') {
                            sh 'mvn clean test'
                        }
                    }
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh 'mvn sonar:sonar'
                }
            }
        }

        stage('Build Frontend') {
            steps {
                dir('frontend') {
                    sh 'npm install'
                    sh 'npm run test'
                    sh 'npm run build'
                }
            }
        }

        stage('Package') {
            steps {
                parallel(
                    "Config Server": {
                        dir('config-server') {
                            sh 'mvn package -DskipTests'
                            sh 'docker build -t microservices/config-server:${BUILD_NUMBER} .'
                        }
                    },
                    "Service Registry": {
                        dir('service-registry') {
                            sh 'mvn package -DskipTests'
                            sh 'docker build -t microservices/service-registry:${BUILD_NUMBER} .'
                        }
                    },
                    "API Gateway": {
                        dir('api-gateway') {
                            sh 'mvn package -DskipTests'
                            sh 'docker build -t microservices/api-gateway:${BUILD_NUMBER} .'
                        }
                    },
                    "Users Service": {
                        dir('microservice-users') {
                            sh 'mvn package -DskipTests'
                            sh 'docker build -t microservices/users-service:${BUILD_NUMBER} .'
                        }
                    },
                    "Products Service": {
                        dir('microservice-products') {
                            sh 'mvn package -DskipTests'
                            sh 'docker build -t microservices/products-service:${BUILD_NUMBER} .'
                        }
                    },
                    "Frontend": {
                        dir('frontend') {
                            sh 'docker build -t microservices/frontend:${BUILD_NUMBER} .'
                        }
                    }
                )
            }
        }

        stage('Push to Registry') {
            steps {
                withCredentials([string(credentialsId: 'docker-registry', variable: 'DOCKER_REGISTRY')]) {
                    sh '''
                        docker tag microservices/config-server:${BUILD_NUMBER} ${DOCKER_REGISTRY}/microservices/config-server:${BUILD_NUMBER}
                        docker tag microservices/service-registry:${BUILD_NUMBER} ${DOCKER_REGISTRY}/microservices/service-registry:${BUILD_NUMBER}
                        docker tag microservices/api-gateway:${BUILD_NUMBER} ${DOCKER_REGISTRY}/microservices/api-gateway:${BUILD_NUMBER}
                        docker tag microservices/users-service:${BUILD_NUMBER} ${DOCKER_REGISTRY}/microservices/users-service:${BUILD_NUMBER}
                        docker tag microservices/products-service:${BUILD_NUMBER} ${DOCKER_REGISTRY}/microservices/products-service:${BUILD_NUMBER}
                        docker tag microservices/frontend:${BUILD_NUMBER} ${DOCKER_REGISTRY}/microservices/frontend:${BUILD_NUMBER}

                        docker push ${DOCKER_REGISTRY}/microservices/config-server:${BUILD_NUMBER}
                        docker push ${DOCKER_REGISTRY}/microservices/service-registry:${BUILD_NUMBER}
                        docker push ${DOCKER_REGISTRY}/microservices/api-gateway:${BUILD_NUMBER}
                        docker push ${DOCKER_REGISTRY}/microservices/users-service:${BUILD_NUMBER}
                        docker push ${DOCKER_REGISTRY}/microservices/products-service:${BUILD_NUMBER}
                        docker push ${DOCKER_REGISTRY}/microservices/frontend:${BUILD_NUMBER}
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withKubeConfig([credentialsId: 'kubeconfig']) {
                    sh '''
                        # Update Kubernetes manifests with new image tags
                        sed -i "s|image: microservices/config-server:.*|image: ${DOCKER_REGISTRY}/microservices/config-server:${BUILD_NUMBER}|g" kubernetes/config-server.yaml
                        sed -i "s|image: microservices/service-registry:.*|image: ${DOCKER_REGISTRY}/microservices/service-registry:${BUILD_NUMBER}|g" kubernetes/service-registry.yaml
                        sed -i "s|image: microservices/api-gateway:.*|image: ${DOCKER_REGISTRY}/microservices/api-gateway:${BUILD_NUMBER}|g" kubernetes/api-gateway.yaml
                        sed -i "s|image: microservices/users-service:.*|image: ${DOCKER_REGISTRY}/microservices/users-service:${BUILD_NUMBER}|g" kubernetes/users-service.yaml
                        sed -i "s|image: microservices/products-service:.*|image: ${DOCKER_REGISTRY}/microservices/products-service:${BUILD_NUMBER}|g" kubernetes/products-service.yaml
                        sed -i "s|image: microservices/frontend:.*|image: ${DOCKER_REGISTRY}/microservices/frontend:${BUILD_NUMBER}|g" kubernetes/frontend.yaml

                        # Apply Kubernetes manifests
                        kubectl apply -f kubernetes/namespace.yaml
                        kubectl apply -f kubernetes/config-server.yaml
                        kubectl apply -f kubernetes/service-registry.yaml
                        kubectl apply -f kubernetes/api-gateway.yaml
                        kubectl apply -f kubernetes/users-service.yaml
                        kubectl apply -f kubernetes/products-service.yaml
                        kubectl apply -f kubernetes/frontend.yaml
                    '''
                }
            }
        }

        stage('Run Integration Tests') {
            steps {
                sh 'mvn verify -Pintegration-tests'
            }
        }

        stage('Monitoring Setup') {
            steps {
                withKubeConfig([credentialsId: 'kubeconfig']) {
                    sh '''
                        kubectl apply -f monitoring/prometheus-config.yaml
                        kubectl apply -f monitoring/grafana-config.yaml
                    '''
                }
            }
        }
    }

    post {
        always {
            junit '**/target/surefire-reports/*.xml'
            archiveArtifacts artifacts: '**/target/*.jar', fingerprint: true

            // Clean up Docker images
            sh '''
                docker rmi microservices/config-server:${BUILD_NUMBER} || true
                docker rmi microservices/service-registry:${BUILD_NUMBER} || true
                docker rmi microservices/api-gateway:${BUILD_NUMBER} || true
                docker rmi microservices/users-service:${BUILD_NUMBER} || true
                docker rmi microservices/products-service:${BUILD_NUMBER} || true
                docker rmi microservices/frontend:${BUILD_NUMBER} || true
            '''
        }

        success {
            echo 'Pipeline executed successfully!'
            slackSend(color: 'good', message: "Build Succeeded: ${env.JOB_NAME} ${env.BUILD_NUMBER}")
        }

        failure {
            echo 'Pipeline execution failed!'
            slackSend(color: 'danger', message: "Build Failed: ${env.JOB_NAME} ${env.BUILD_NUMBER}")
        }
    }
}