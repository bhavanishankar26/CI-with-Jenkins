pipeline {
    agent any

    environment {
        NAME = "spring-app"
        VERSION = "${env.BUILD_ID}"
        IMAGE_REPO = "indalarajesh"
        GIT_REPO_NAME = "CI-with-Jenkins"
        GIT_USER_NAME = "INDALARAJESH"
    }

    tools { 
        maven 'maven-3.8.6' // Ensure maven-3.8.6 is properly configured in Jenkins
    }

    stages {
        stage('Install Docker') {
            steps {
                sh '''
                sudo apt update -y
                sudo apt install docker.io -y
                sudo chmod 666 /var/run/docker.sock
                '''
            }
        }

        stage('Checkout git') {
            steps {
                git branch: 'main', url: 'https://github.com/indalarajesh/CI-with-Jenkins.git'
            }
        }

        stage('Get Git Commit Hash') {
            steps {
                script {
                    env.GIT_COMMIT = sh(returnStdout: true, script: 'git rev-parse HEAD').trim()
                }
            }
        }
        
        stage ('Build & JUnit Test') {
            steps {
                sh 'mvn clean install'
            }
        }
        
        stage('Docker Build') {
            steps {
                sh 'docker build -t ${IMAGE_REPO}/${NAME}:${VERSION}-${GIT_COMMIT} .'
            }
        }
        
        stage('Docker Image Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-credentials-id', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                    sh "docker login -u ${DOCKER_USERNAME} -p ${DOCKER_PASSWORD}"
                    sh 'docker push ${IMAGE_REPO}/${NAME}:${VERSION}-${GIT_COMMIT}'
                    sh 'docker rmi ${IMAGE_REPO}/${NAME}:${VERSION}-${GIT_COMMIT}'
                }
            }
        }
        
        stage('Clone/Pull k8s deployment Repo') {
            steps {
                script {
                    if (fileExists('CI-with-Jenkins')) {
                        echo 'Cloned repo already exists - Pulling latest changes'
                        dir("CI-with-Jenkins") {
                            sh 'git pull'
                        }
                    } else {
                        echo 'Repo does not exist - Cloning the repo'
                        sh 'git clone -b feature https://github.com/INDALARAJESH/CI-with-Jenkins.git'
                    }
                }
            }
        }
        
        stage('Update deployment Manifest') {
            steps {
                dir("CI-with-Jenkins/yamls") {
                    sh 'sed -i "s#indalarajesh.*#${IMAGE_REPO}/${NAME}:${VERSION}-${GIT_COMMIT}#g" deployment.yaml'
                    sh 'cat deployment.yaml'
                }
            }
        }
        
        stage('Commit & Push changes to feature branch') {
            steps {
                withCredentials([string(credentialsId: 'GITHUB_TOKEN', variable: 'GITHUB_TOKEN')]) {
                    dir("CI-with-Jenkins/yamls") {
                        sh "git config --global user.email 'rajeshindala1997@gmail.com'"
                        sh "git push https://x-access-token:${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME}.git feature"
                        sh 'git checkout feature'
                        sh 'git add deployment.yaml'
                        sh "git commit -am 'Updated image version for Build- ${VERSION}-${GIT_COMMIT}'"
                        sh 'git push origin feature'
                    }
                }    
            }
        }
        
        stage('Raise PR') {
            steps {
                withCredentials([string(credentialsId: 'GITHUB_TOKEN', variable: 'GITHUB_TOKEN')]) {
                    dir("CI-with-Jenkins/yamls") {
                        sh 'echo "${GITHUB_TOKEN}" | gh auth login --with-token'
                        sh 'git checkout feature'
                        sh "gh pr create -t 'Image tag
