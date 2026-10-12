pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        timeout(time: 10, unit: 'MINUTES')
    }

    triggers {
        pollSCM('H/2 * * * *')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    python3 -m venv .venv
                    . .venv/bin/activate
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

        stage('Build') {
            steps {
                sh 'docker build -t interview-app:${BUILD_NUMBER} .'
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
            script {
                def successful =
                    currentBuild.currentResult == 'SUCCESS' ? 1 : 0
                def durationSeconds =
                    (currentBuild.duration ?: 0) / 1000.0

                withEnv([
                    "PIPELINE_BUILD_STATUS=${successful}",
                    "PIPELINE_BUILD_DURATION=${durationSeconds}"
                ]) {
                    try {
                        sh '''
                            python3 - <<'PY'
import os
import xml.etree.ElementTree as ET

passed = 0
failed = 0
report = "reports/junit.xml"

if os.path.exists(report):
    root = ET.parse(report).getroot()

    for case in root.iter("testcase"):
        if case.find("failure") is not None or case.find("error") is not None:
            failed += 1
        elif case.find("skipped") is None:
            passed += 1

with open("ci-metrics.txt", "w") as metrics:
    metrics.write(
        f"pipeline_build_status {os.environ['PIPELINE_BUILD_STATUS']}\\n"
    )
    metrics.write(
        f"pipeline_build_duration_seconds "
        f"{os.environ['PIPELINE_BUILD_DURATION']}\\n"
    )
    metrics.write(f"pipeline_tests_passed {passed}\\n")
    metrics.write(f"pipeline_tests_failed {failed}\\n")
PY

                            curl --fail --silent --show-error \
                                --data-binary @ci-metrics.txt \
                                http://pushgateway:9091/metrics/job/interview_pipeline
                        '''
                    } finally {
                        sh 'docker rm -f interview-app-test || true'
                        deleteDir()
                    }
                }
            }
        }
    }
}