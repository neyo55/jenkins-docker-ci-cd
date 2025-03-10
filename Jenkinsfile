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
                    docker run -d -p 5000:5000 --name test_container neyo55/jenkins-docker-ci-cd:latest
                '''
            }
        }
    }

    post {
        success {
            emailext subject: "Jenkins Build SUCCESS: ${env.JOB_NAME}",
                     body: "The build was successful. Check it here: ${env.BUILD_URL}",
                     to: "kbneyo55@gmail.com"
        }

        failure {
            emailext subject: "Jenkins Build FAILED: ${env.JOB_NAME}",
                     body: "The build failed. Check logs here: ${env.BUILD_URL}",
                     to: "kbneyo55@gmail.com"
        }
    }
}
