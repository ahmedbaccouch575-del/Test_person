pipeline {
    agent any

    environment {
        IMAGE  = "ahmed0020"   // À CHANGER
        EC2_IP = "172.31.35.247"                  // À CHANGER (sans ubuntu@)
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Test') {
            steps {
                bat '''
                    python -m venv venv
                    call venv\\Scripts\\activate
                    pip install -r requirements.txt
                    pytest -v
                '''
            }
        }

        stage('Build Docker image') {
            steps {
                bat 'docker build -t %IMAGE%:%BUILD_NUMBER% -t %IMAGE%:latest .'
            }
        }

        stage('Push Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub',   // À cHANGER si autre ID
                        usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
                    bat '''
                        echo %DH_PASS%| docker login -u %DH_USER% --password-stdin
                        docker push %IMAGE%:%BUILD_NUMBER%
                        docker push %IMAGE%:latest
                    '''
                }
            }
        }

        stage('Deploy AWS') {
            steps {
                withCredentials([file(credentialsId: 'ec2-key', variable: 'KEY')]) {   // À CHANGER si autre ID
                    bat '''
                        ssh -o StrictHostKeyChecking=no -i "%KEY%" ubuntu@%EC2_IP% "docker pull %IMAGE%:latest && (docker rm -f cicd-demo || true) && docker run -d --name cicd-demo -p 5000:5000 %IMAGE%:latest"
                    '''
                }
            }
        }

        stage('Smoke test') {
            steps {
                bat '''
                    ping -n 10 127.0.0.1 > nul
                    curl -f http://%EC2_IP%:5000/health
                '''
            }
        }
    }

    post {
        success { echo 'Pipeline OK : application déployée sur AWS' }
        failure { echo 'Pipeline en échec' }
    }
}
