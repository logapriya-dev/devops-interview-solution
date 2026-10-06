pipeline {
    agent any

    options {
        timeout(time: 10, unit: 'MINUTES')
    }

    stages {

        stage('Install Dependencies') {
            steps {
                sh '''
                    python3 -m venv .venv
                    . .venv/bin/activate
                    pip install --upgrade pip
                    pip install -r app/requirements.txt
                '''
            }
        }

        stage('Lint') {
            steps {
                sh '''
                    . .venv/bin/activate
                    flake8 app/
                '''
            }
        }

        stage('Test') {
            steps {
                sh '''
                    . .venv/bin/activate
                    mkdir -p reports
                    pytest --junitxml=reports/junit.xml
                '''
            }
            post {
                always {
                    junit 'reports/junit.xml'
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    docker build -t interview-app:${BUILD_NUMBER} .
                '''
            }
        }

        stage('Smoke Test') {
            steps {
                sh '''
                    docker rm -f interview-app-test || true

                    docker run -d \
                        --name interview-app-test \
                        --add-host=host.docker.internal:host-gateway \
                        -p 5000:5000 \
                        interview-app:${BUILD_NUMBER}

                    sleep 5

                    curl --fail http://host.docker.internal:5000/health
                '''
            }
        }
    }

    post {
        always {
            sh '''
                docker rm -f interview-app-test || true
            '''
        }
    }
}