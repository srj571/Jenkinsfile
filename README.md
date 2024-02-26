# Jenkinsfile
pipeline {
    agent any
    environment {
        DOCKERHUB_CREDENTIALS = credentials('Docker_Credentials')
        IMAGE_TAG = "${BUILD_NUMBER}"
        DOCKER_IMAGE = "surajm2w/servicepoints"
    }

    stages {
        stage('Checkout Code') {
            steps {
                checkout scmGit(branches: [[name: '*/devOps_production']], extensions: [], userRemoteConfigs: [[credentialsId: 'github_token_sp', url: 'https://github.com/rohitmind2web/dropshipping.git']])
            }
        }
        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${DOCKER_IMAGE}:${IMAGE_TAG} ."
            }
        }
        stage('Docker Login') {
            steps {
                sh "echo '$DOCKERHUB_CREDENTIALS_PSW' | docker login -u '$DOCKERHUB_CREDENTIALS_USR' --password-stdin"
            }
        }
        stage('Push Image') {
            steps {
                sh "docker tag ${DOCKER_IMAGE}:${IMAGE_TAG} ${DOCKER_IMAGE}:latest"
                sh "docker push ${DOCKER_IMAGE}:${IMAGE_TAG}"
                sh "docker push ${DOCKER_IMAGE}:latest"
            }
        }
        stage('Deploy to k8s') {
            steps {
                script {
                    sh "sed -i 's/latest/${IMAGE_TAG}/g' 'Kubernetes file/webapp.yaml'"
                    kubernetesDeploy configs: 'Kubernetes file/webapp.yaml', kubeConfig: [path: ''], kubeconfigId: 'k8sconfig', secretName: '', ssh: [sshCredentialsId: '*', sshServer: ''], textCredentials: [certificateAuthorityData: '', clientCertificateData: '', clientKeyData: '', serverUrl: 'https://']
                    
                    sh "sed -i 's/latest/${IMAGE_TAG}/g' 'Kubernetes file/17track_webhook.yaml'"
                    kubernetesDeploy configs: 'Kubernetes file/17track_webhook.yaml', kubeConfig: [path: ''], kubeconfigId: 'k8sconfig', secretName: '', ssh: [sshCredentialsId: '*', sshServer: ''], textCredentials: [certificateAuthorityData: '', clientCertificateData: '', clientKeyData: '', serverUrl: 'https://']
                }
            }
        }
    }
    post {
        always {
            sh "docker rmi ${DOCKER_IMAGE}:${IMAGE_TAG}"
            sh "docker rmi ${DOCKER_IMAGE}:latest"
            sh 'docker logout'
        }
    }
}
