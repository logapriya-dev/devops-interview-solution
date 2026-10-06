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
            script {
                def buildStatus = currentBuild.currentResult ?: 'UNKNOWN'
                def durationSeconds = (currentBuild.duration ?: 0) / 1000.0

                withEnv([
                    "CI_BUILD_STATUS=${buildStatus}",
                    "CI_BUILD_DURATION=${durationSeconds}"
                ]) {
                    sh '''
                        total=0
                        passed=0
                        failed=0
                        skipped=0
                        errors=0

                        if [ -f reports/junit.xml ]; then
                            total=$(python3 -c 'import xml.etree.ElementTree as ET; r=ET.parse("reports/junit.xml").getroot(); s=[r] if r.tag=="testsuite" else r.findall(".//testsuite"); print(sum(int(x.attrib.get("tests",0)) for x in s))')
                            failed=$(python3 -c 'import xml.etree.ElementTree as ET; r=ET.parse("reports/junit.xml").getroot(); s=[r] if r.tag=="testsuite" else r.findall(".//testsuite"); print(sum(int(x.attrib.get("failures",0)) for x in s))')
                            skipped=$(python3 -c 'import xml.etree.ElementTree as ET; r=ET.parse("reports/junit.xml").getroot(); s=[r] if r.tag=="testsuite" else r.findall(".//testsuite"); print(sum(int(x.attrib.get("skipped",0)) for x in s))')
                            errors=$(python3 -c 'import xml.etree.ElementTree as ET; r=ET.parse("reports/junit.xml").getroot(); s=[r] if r.tag=="testsuite" else r.findall(".//testsuite"); print(sum(int(x.attrib.get("errors",0)) for x in s))')
                            passed=$((total - failed - skipped - errors))
                        fi

                        cat > ci-metrics.txt <<EOF
ci_build_status{status="${CI_BUILD_STATUS}"} 1
ci_build_duration_seconds ${CI_BUILD_DURATION}
ci_tests_total ${total}
ci_tests_passed ${passed}
ci_tests_failed ${failed}
ci_tests_skipped ${skipped}
EOF

                        curl --fail \
                            --data-binary @ci-metrics.txt \
                            http://pushgateway:9091/metrics/job/devops_interview/build/${BUILD_NUMBER}
                    '''
                }
            }

            sh '''
                docker rm -f interview-app-test || true
            '''
        }
    }
}