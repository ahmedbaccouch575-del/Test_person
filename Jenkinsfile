pipeline {
    agent any

    options {
        timestamps()
        timeout(time: 20, unit: 'MINUTES')
    }

    environment {
        IMAGE  = "ahmed0020/cicd-demo"
        EC2_IP = "13.53.73.77"   // IPv4 publique de l'EC2 (pas 172.31.x.x)
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Test') {
            steps {
                sh '''
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install -r requirements.txt
                    pytest -v
                '''
            }
        }

        stage('Check Docker') {
            steps {
                sh 'docker --version'
                sh 'docker info --format "{{.ServerVersion}}"'
            }
        }

        stage('Build Docker image') {
            steps {
                sh 'docker build -t $IMAGE:$BUILD_NUMBER -t $IMAGE:latest .'
            }
        }

        stage('Push Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
                    sh '''
                        echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
                        docker push $IMAGE:$BUILD_NUMBER
                        docker push $IMAGE:latest
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy AWS') {
            steps {
                withCredentials([file(credentialsId: 'ec2-key', variable: 'KEY')]) {
                    sh '''
                        chmod 600 "$KEY"
                        ssh -o StrictHostKeyChecking=no -i "$KEY" ubuntu@$EC2_IP \
                          "docker pull $IMAGE:latest && (docker rm -f cicd-demo || true) && docker run -d --name cicd-demo -p 5000:5000 $IMAGE:latest"
                    '''
                }
            }
        }

        stage('Smoke test') {
            steps {
                retry(5) {
                    sleep time: 5, unit: 'SECONDS'
                    sh 'curl -f http://$EC2_IP:5000/health'
                }
            }
        }
    }

    post {
        success { echo 'Pipeline OK : application déployée sur AWS' }
        failure { echo 'Pipeline en échec' }
    }
}
