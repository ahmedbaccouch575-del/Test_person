pipeline {
    agent any

    options {
        timestamps()
        timeout(time: 20, unit: 'MINUTES')
    }

    environment {
        // IMPORTANT : pour Docker Hub, le nom doit être  utilisateur/repo
        IMAGE  = "ahmed0020/cicd-demo"      // À CHANGER (ton user Docker Hub / nom du repo)
        EC2_IP = "172.31.35.247"                  // À CHANGER : IP PUBLIQUE de l'EC2
        PYTHON = "C:\\Users\\ahmed\\AppData\\Local\\Programs\\Python\\Python312\\python.exe"
        DOCKER = "wsl docker"               // si Docker Desktop : mets "docker"
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
                    "%PYTHON%" -m venv venv
                    call venv\\Scripts\\activate
                    pip install -r requirements.txt
                    pytest -v
                '''
            }
        }

        stage('Check Docker') {
            steps {
                bat '%DOCKER% --version'
                bat '%DOCKER% info --format "{{.ServerVersion}}"'
            }
        }

        stage('Build Docker image') {
            steps {
                bat '%DOCKER% build -t %IMAGE%:%BUILD_NUMBER% -t %IMAGE%:latest .'
            }
        }

        stage('Push Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub',
                        usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
                    bat '''
                        echo %DH_PASS%| %DOCKER% login -u %DH_USER% --password-stdin
                        %DOCKER% push %IMAGE%:%BUILD_NUMBER%
                        %DOCKER% push %IMAGE%:latest
                        %DOCKER% logout
                    '''
                }
            }
        }

        stage('Deploy AWS') {
            steps {
                withCredentials([file(credentialsId: 'ec2-key', variable: 'KEY')]) {
                    bat '''
                        icacls "%KEY%" /inheritance:r
                        icacls "%KEY%" /grant:r "%USERNAME%:R"
                        ssh -o StrictHostKeyChecking=no -i "%KEY%" ubuntu@%EC2_IP% "docker pull %IMAGE%:latest && (docker rm -f cicd-demo || true) && docker run -d --name cicd-demo -p 5000:5000 %IMAGE%:latest"
                    '''
                }
            }
        }

        stage('Smoke test') {
            steps {
                retry(5) {
                    sleep time: 5, unit: 'SECONDS'
                    bat 'curl -f http://%EC2_IP%:5000/health'
                }
            }
        }
    }

    post {
        success { echo 'Pipeline OK : application déployée sur AWS' }
        failure { echo 'Pipeline en échec' }
    }
}
