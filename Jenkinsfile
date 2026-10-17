@Library('birametgod-shared-library')
pipeline {
    environment {
       DOCKER_USER = 'birametgod'
       IMAGE_NAME = 'alpinehelloworld'
       IMAGE_TAG = 'latest'
       STAGING_HOST = '51.20.2.253'
       PROD_HOST = '13.53.212.180'
    }
    agent none
    stages {
        stage('Build image') {
            agent any
            steps {
                script {
                    sh 'docker build -t ${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG} .'
                }
                echo 'Building...'
            }
        }
        stage('Run container based on build Image') {
            agent any
            steps {
                script {
                    sh '''
                        docker run -d -p 80:5000 -e PORT=5000 --name ${IMAGE_NAME} ${DOCKER_USER}/${IMAGE_NAME}:${IMAGE_TAG}
                        sleep 5
                    '''
                }
                echo 'Running container...'
            }
        }
        stage('Test image') {
            agent any
            steps {
                script {
                    sh 'curl http://localhost | grep -q "Hello world!"'
                }
                echo 'Testing...'
            }
        }
        stage('Stop container') {
            agent any
            steps {
                script {
                    sh '''
                        docker stop ${IMAGE_NAME}
                        docker rm ${IMAGE_NAME}
                    '''
                }
                echo 'Stopping container...'
            }
        }
        stage('Push image') {
            agent any
            when {
                expression { return env.BRANCH_NAME == 'master' }
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub_birametgod',
                usernameVariable: 'U', passwordVariable: 'P')]) {
                    sh 'echo $P | docker login -u $U --password-stdin'
                    sh 'docker push $DOCKER_USER/$IMAGE_NAME:$IMAGE_TAG'
                }
            }
        }
        stage('Deploy staging') {
            agent any
            when { expression { env.GIT_BRANCH == 'origin/master' } }
            steps {
                sshagent(['deployapp-key']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no ubuntu@$STAGING_HOST "
                        sudo docker pull $DOCKER_USER/$IMAGE_NAME:latest &&
                        (sudo docker rm -f webapp || true) &&
                        sudo docker run -d -p 80:5000 -e PORT=5000 --name webapp $DOCKER_USER/$IMAGE_NAME:latest"
                    '''
                }
            }
        }

        stage('Deploy production') {
            agent any
            when { expression { env.GIT_BRANCH == 'origin/master' } }
            steps {
                sshagent(['deployapp-key']) {
                sh '''
                    ssh -o StrictHostKeyChecking=no ubuntu@$PROD_HOST "
                    sudo docker pull $DOCKER_USER/$IMAGE_NAME:latest &&
                    (sudo docker rm -f webapp || true) &&
                    sudo docker run -d -p 80:5000 -e PORT=5000 --name webapp $DOCKER_USER/$IMAGE_NAME:latest"
                '''
                }
            }
        }
    }

    post {
        always {
            script {
                slackNotifier currentBuild.result 
            }
        }
    }
}