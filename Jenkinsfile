pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "neyo55/jenkins-docker-ci-cd"
    }

    stages {
        stage('Clean Workspace') {
            steps {
                sh 'rm -rf *'
            }
        }

        stage('Clone Repository') {
            steps {
                git branch: 'main', url: 'https://github.com/neyo55/jenkins-docker-ci-cd.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $DOCKER_IMAGE:latest .'
            }
        }

        stage('Run Tests') {
            steps {
                script {
                    def testDirExists = sh(script: 'if [ -d "tests" ]; then echo "exists"; else echo "missing"; fi', returnStdout: true).trim()
                    if (testDirExists == "exists") {
                        sh 'docker run --rm $DOCKER_IMAGE pytest tests/'
                    } else {
                        echo "No tests found, skipping test stage."
                    }
                }
            }
        }        

        stage('Push to Docker Hub') {
            steps {
                withDockerRegistry([credentialsId: 'docker-hub-cred', url: '']) {
                    sh 'docker push $DOCKER_IMAGE:latest'
                }
            }
        }

        stage('Deploy to Test Environment') {
            steps {
                sh '''
                    docker stop test_container || true
                    docker rm test_container || true
                    docker run -d -p 5000:5000 --name test_container $DOCKER_IMAGE:latest
                '''
            }
        }

        stage('Docker Cleanup') {
            steps {
                sh '''
                    echo "Removing unused containers..."
                    docker container prune -f

                    echo "Removing dangling images..."
                    docker image prune -f

                    echo "Removing unused networks..."
                    docker network prune -f

                    echo "Removing unused volumes..."
                    docker volume prune -f
                '''
            }
        }
    }

    post {
        success {
            mail to: 'kbneyo55@gmail.com',
                 subject: "Jenkins Build Success: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                 body: "Good job! The Jenkins build for ${env.JOB_NAME} #${env.BUILD_NUMBER} was successful! Check it out here: ${env.BUILD_URL}"
        }
        failure {
            mail to: 'kbneyo55@gmail.com',
                 subject: "Jenkins Build Failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                 body: "Oops! The Jenkins build for ${env.JOB_NAME} #${env.BUILD_NUMBER} failed. Please check the logs: ${env.BUILD_URL}",
        }
    }
}
