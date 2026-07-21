pipeline {
    agent any

    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/ramkiprabhu/Tetris-V1.git'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''
                    $SCANNER_HOME/bin/sonar-scanner \
                    -Dsonar.projectName=Tetris \
                    -Dsonar.projectKey=Tetris
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                dependencyCheck(
                    additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit --failOnError false',
                    odcInstallation: 'DP-Check'
                )

                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('Trivy File System Scan') {
            steps {
                sh 'trivy fs . > trivyfs.txt'
            }
        }

        stage('Docker Build & Push') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'docker',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh '''
                    echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin

                    docker build -t tetris .

                    docker tag tetris ${DOCKER_USER}/tetris:latest

                    docker push ${DOCKER_USER}/tetris:latest

                    docker logout
                    '''
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh 'trivy image ramkiprabhu28/tetris:latest > trivyimage.txt'
            }
        }

        stage('Deploy Docker Container') {
            steps {
                sh '''
                docker rm -f tetris || true

                docker run -d \
                --name tetris \
                -p 3000:3000 \
                ramkiprabhu28/tetris:latest
                '''
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                dir('Jenkins-terraform') {
                    withKubeConfig(
                        credentialsId: 'K8S',
                        restrictKubeConfigAccess: false
                    ) {
                        sh 'kubectl apply -f deployment-service.yml'
                    }
                }
            }
        }
    }
}

