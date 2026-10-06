\# DevOps Interview Solution - Notes



\## 1. Pipeline Design



The CI/CD pipeline is implemented using Jenkins Pipeline as Code.



Pipeline flow:



GitHub

→ Jenkins

→ Install Dependencies

→ Flake8 Lint

→ Pytest

→ JUnit Test Report

→ Docker Build

→ Docker Smoke Test

→ Push CI Metrics

→ Cleanup



The pipeline has a 10-minute timeout.



\## 2. Test Fix



The original `/add` test expected HTTP status code 201.



The Flask endpoint returns HTTP 200 for a successful GET request, so the test expectation was corrected from 201 to 200.



After the fix:



5 tests passed successfully.



\## 3. Failure Handling



The pipeline is sequential.



Docker build and smoke testing do not run when earlier stages fail.



JUnit results are published after the test stage.



Temporary Docker containers are removed in the post-build cleanup block using:



docker rm -f interview-app-test || true



\## 4. Docker



The Flask application is packaged as a Docker image.



The pipeline creates an image using:



interview-app:${BUILD\_NUMBER}



The container exposes port 5000.



A smoke test verifies:



/health



Expected response:



{"status":"ok"}



\## 5. Monitoring



Jenkins pushes CI metrics to Pushgateway.



Pushgateway is scraped by Prometheus.



Grafana uses Prometheus as its data source.



Metrics currently collected:



\- ci\_build\_status

\- ci\_build\_duration\_seconds

\- ci\_tests\_total

\- ci\_tests\_passed

\- ci\_tests\_failed

\- ci\_tests\_skipped



\## 6. Grafana Dashboard



The dashboard displays:



\- CI Build Status

\- Build Duration

\- Tests Passed

\- Tests Failed

\- Total Tests



The latest successful pipeline produced:



\- Build status: SUCCESS

\- Tests total: 5

\- Tests passed: 5

\- Tests failed: 0

\- Tests skipped: 0

\- Build duration: approximately 29.7 seconds



\## 7. Production Improvements



For a production environment, I would improve the solution by:



\- Using Jenkins credentials or a secret manager instead of public repositories where appropriate.

\- Using Docker BuildKit/buildx instead of the legacy Docker builder.

\- Pinning and regularly updating base images.

\- Adding container vulnerability scanning.

\- Adding artifact/image versioning and registry push.

\- Running Jenkins agents with least privilege rather than using the Docker socket directly.

\- Adding alerts for failed builds and metric anomalies.

\- Keeping Grafana dashboards and provisioning configuration under version control.



\## 8. Local Interview Environment



The provided Docker Compose environment runs:



\- Jenkins on port 8080

\- Pushgateway on port 9091

\- Prometheus on port 9090

\- Grafana on port 3000

