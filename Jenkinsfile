pipeline {
    agent any

    environment {
        NAME = "spring-app"
        VERSION = "${env.BUILD_ID}"
        IMAGE_REPO = "gadebhavani26"
        GIT_REPO_NAME = "CD-k8s"
        GIT_USER_NAME = "bhavanishankar26"
        AWS_REGION = "us-east-1" // Set your AWS region
        EKS_CLUSTER_NAME = "terraform-eks-demo" // Set your EKS cluster name
    }

    tools { 
        maven 'maven-3.8.6' 
    }

    stages {
        stage('Checkout git') {
            steps {
                git branch: 'main', url: 'https://github.com/bhavanishankar26/CI-with-Jenkins.git'
            }
        }

        stage('Get Git Commit Hash') {
            steps {
                script {
                    env.GIT_COMMIT = sh(returnStdout: true, script: 'git rev-parse HEAD').trim()
                }
            }
        }
        
        stage('Build & JUnit Test') {
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
                    if (fileExists('CD-k8s')) {
                        echo 'Cloned repo already exists - Pulling latest changes'
                        dir("CD-k8s") {
                            sh 'git pull'
                        }
                    } else {
                        echo 'Repo does not exist - Cloning the repo'
                        sh 'git clone -b feature https://github.com/bhavanishankar26/CD-k8s.git'
                    }
                }
            }
        }
        
        stage('Update deployment Manifest') {
            steps {
                dir("CD-k8s/yamls") {
                    sh 'sed -i "s#\\(image: \\).*#\\1${IMAGE_REPO}/${NAME}:${VERSION}-${GIT_COMMIT}#g" deployment.yaml'
                    sh 'cat deployment.yaml'
                }
            }
        }
        
        stage('Commit & Push changes to feature branch') {
            steps {
                withCredentials([string(credentialsId: 'GITHUB_TOKEN', variable: 'GITHUB_TOKEN')]) {
                    dir("CD-k8s/yamls") {
                        sh "git config --global user.email 'gadebhavani26@gmail.com'"
                        sh "git config --global user.name 'gadebhavani26'"
                        sh 'git remote set-url origin https://${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME}'
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
                    dir("CD-k8s/yamls") {
                        sh '''
                            unset GITHUB_TOKEN
                            echo "${GITHUB_TOKEN}" | gh auth login --with-token
                        '''
                        sh 'git checkout feature'
                        sh "gh pr create -t 'Image tag updated' -b 'Check and merge it'"
                    }
                }
            }
        }

        stage('Configure AWS CLI') {
            steps {
                script {
                    withCredentials([
                        string(credentialsId: 'aws-access-key-id', variable: 'AWS_ACCESS_KEY_ID'), 
                        string(credentialsId: 'aws-secret-access-key', variable: 'AWS_SECRET_ACCESS_KEY')
                    ]) {
                        sh '''
                            aws configure set aws_access_key_id ${AWS_ACCESS_KEY_ID}
                            aws configure set aws_secret_access_key ${AWS_SECRET_ACCESS_KEY}
                            aws configure set default.region ${AWS_REGION}
                        '''
                    }
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                script {
                    // Configure kubectl to use your EKS cluster
                    sh "aws eks --region ${AWS_REGION} update-kubeconfig --name ${EKS_CLUSTER_NAME}"

                    // Deploy to EKS using kubectl
                    sh 'kubectl apply -f CD-k8s/yamls/deployment.yaml --validate=false'
                    sh 'kubectl apply -f CD-k8s/yamls/service.yaml --validate=false'
                }
            }
        }
    }
}
