
pipeline {
    agent {
        docker {
            image 'maven:3.9-eclipse-temurin-21'
            args '--user root -v /var/run/docker.sock:/var/run/docker.sock'
        }
    }

    stages {
        stage('Checkout') {
            steps {
                sh 'echo passed'
                git branch: 'main', url: 'https://github.com/Siddhi-Panadi30/CI-CD-Pipeline-With-Observability.git'
            }
        }

        stage('Backend Build and Test') {
            steps {
                sh 'cd backend && mvn clean package'
            }
        }

        stage('Frontend Build') {
            steps {
                sh 'cd netflix-clone && npm install'
                sh 'npm run build'
            }
        }

        stage('Static Code Analysis') {
            environment {
                SONAR_URL = 'http://32.202.199.45:9000'
            }

            steps {
                withCredentials([
                    string(
                        credentialsId: 'sonarqube',
                        variable: 'SONAR_AUTH_TOKEN'
                    )
                ]) {

                    // Backend analysis
                    sh '''
                        cd backend

                        mvn sonar:sonar \
                            -Dsonar.projectKey=netflix-backend \
                            -Dsonar.projectName="Netflix Backend" \
                            -Dsonar.token=$SONAR_AUTH_TOKEN \
                            -Dsonar.host.url=$SONAR_URL
                    '''

                    // Frontend analysis
                    sh '''
                        cd netflix-clone

                        sonar-scanner \
                            -Dsonar.projectKey=netflix-frontend \
                            -Dsonar.projectName="Netflix Frontend" \
                            -Dsonar.sources=src \
                            -Dsonar.exclusions=node_modules/**,build/**,dist/** \
                            -Dsonar.token=$SONAR_AUTH_TOKEN \
                            -Dsonar.host.url=$SONAR_URL
                    '''
                }
            }
        }

        stage('Build and Push Docker Image') {
            environment {
                DOCKER_BACKEND_IMAGE = 'siddhip30/netflix-backend'
                DOCKER_FRONTEND_IMAGE = 'siddhip30/netflix-frontend'
                REGISTRY_CREDENTIALS = credentials('docker-cred')
            }

            steps {
                script {
                    sh 'cd netflix-clone/backend && docker build -t ${DOCKER_BACKEND_IMAGE} .'
                    sh 'cd netflix-clone && docker build -t ${DOCKER_FRONTEND_IMAGE}.'

                    def backendImage = docker.image("${DOCKER_BACKEND_IMAGE}")
                    def frontendImage = docker.image("${DOCKER_FRONTEND_IMAGE}")

                    docker.withRegistry('https://index.docker.io/v1/', "docker-cred") {
                        backendImage.push()
                        frontendImage.push()
                    }
                }
            }
        }

        stage('Update Deployment File') {
            environment {
                GIT_REPO_NAME = "CI-CD-Pipeline-With-Observability"
                GIT_USER_NAME = "Siddhi-Panadi30"
            }

            steps {
                withCredentials([
                    string(
                        credentialsId: 'github',
                        variable: 'GITHUB_TOKEN'
                    )
                ]) {
                    sh '''
                        git config user.email "siddhipanadi@gmail.com"
                        git config user.name "Siddhi-Panadi30"

                        BUILD_NUMBER=${BUILD_NUMBER}

                        sed -i "s/replaceImageTag/${BUILD_NUMBER}/g" netflix-clone/k8s/backend/deployment.yaml
                        sed -i "s|replaceImageTag|${BUILD_NUMBER}|g" netflix-clone/k8s/frontend/deployment.yaml

                        git add netflix-clone/k8s/backend/deployment.yaml
                        git add netflix-clone/k8s/frontend/deployment.yaml

                        git commit -m "Update deployment image to version ${BUILD_NUMBER}"

                        git push https://${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME} HEAD:main
                    '''
                }
            }
        }
    }
}




