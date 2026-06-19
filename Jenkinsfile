pipeline {
    agent any
    tools {
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        APP_NAME     = "netflix-clone"
        DOCKERHUB_USER = "shravani1608"
    }
    stages {

        stage('Git Checkout') {
            steps {
                git branch: 'main',
                    credentialsId: 'github-token',
                    url: 'https://github.com/Shravani1603/Netflix-clone.git'
                echo "✅ Code checkout done"
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
                echo "✅ npm install done"
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=netflix-clone \
                        -Dsonar.projectKey=netflix-clone
                    '''
                }
                echo "✅ SonarQube analysis done"
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false
                }
                echo "✅ Quality gate checked"
            }
        }

        stage('Docker Build') {
            steps {
                sh """
                    docker build \
                    --build-arg TMDB_V3_API_KEY=441a3ce6579b398ae3511bea29f28a4c \
                    -t ${APP_NAME}:latest \
                    .
                """
                echo "✅ Docker image built"
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh """
                    trivy image \
                    --format table \
                    --severity HIGH,CRITICAL \
                    --output trivy-report.txt \
                    ${APP_NAME}:latest || true
                """
                echo "✅ Trivy scan done"
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivy-report.txt',
                    allowEmptyArchive: true
                }
            }
        }

        stage('Push to DockerHub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-cred',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh """
                        echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                        docker tag ${APP_NAME}:latest ${DOCKERHUB_USER}/${APP_NAME}:latest
                        docker push ${DOCKERHUB_USER}/${APP_NAME}:latest
                    """
                }
                echo "✅ Image pushed to DockerHub"
            }
        }

        stage('Deploy') {
            steps {
                sh """
                    docker stop netflix-app 2>/dev/null || true
                    docker rm netflix-app 2>/dev/null || true
                    docker run -d \
                        --name netflix-app \
                        -p 80:80 \
                        ${DOCKERHUB_USER}/${APP_NAME}:latest
                """
                echo "✅ App deployed!"
            }
        }
    }

    post {
        success {
            echo "🎉 Pipeline SUCCESS!"
        }
        failure {
            echo "❌ Pipeline FAILED - Check console output"
        }
        always {
            cleanWs()
        }
    }
}
