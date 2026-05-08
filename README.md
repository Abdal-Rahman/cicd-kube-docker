# CI/CD Pipeline: Docker + Kubernetes + Jenkins

An end-to-end CI/CD pipeline that builds a Java web application, containerizes it with Docker, runs automated quality gates, and deploys to a Kubernetes cluster using Helm charts.

## Architecture

```
Developer Push
      │
      ▼
  Jenkins Pipeline
      │
      ├── 1. Build           (Maven → WAR artifact)
      ├── 2. Unit Tests      (JUnit via Maven)
      ├── 3. Integration Tests
      ├── 4. Code Analysis   (Checkstyle + SonarQube)
      ├── 5. Docker Build    (Tomcat 8 + JRE 11 image)
      ├── 6. Push to DockerHub
      └── 7. Helm Deploy → Kubernetes (production namespace)
```

## Tech Stack

| Layer | Technology |
|---|---|
| Application | Java, Spring MVC, Maven |
| Containerization | Docker, Tomcat 8 / JRE 11 |
| CI/CD | Jenkins (declarative pipeline) |
| Container Registry | DockerHub |
| Orchestration | Kubernetes (KOPS) |
| Packaging | Helm |
| Code Quality | SonarQube, Checkstyle |

## Repository Structure

```
├── src/                    # Java application source code
├── helm/
│   └── vprofilecharts/     # Helm chart for Kubernetes deployment
│       ├── templates/      # K8s manifest templates
│       ├── Chart.yaml      # Chart metadata
│       └── values.yaml     # Configurable deployment values
├── kubernetes/             # Raw Kubernetes manifests
├── Dockerfile              # Container image definition
├── Jenkinsfile             # Declarative pipeline definition
└── pom.xml                 # Maven build configuration
```

## Pipeline Breakdown

### Stage 1 — Build
Compiles source code and packages it into a WAR artifact using Maven, skipping tests for speed.

### Stage 2 & 3 — Testing
Runs unit tests with JUnit and integration tests separately, ensuring both layers of validation pass before proceeding.

### Stage 4 — Code Quality
- **Checkstyle**: enforces coding standards
- **SonarQube**: static analysis for bugs, vulnerabilities, and code smells

### Stage 5 & 6 — Dockerize & Push
Builds a Docker image tagged with the Jenkins build number (versioned releases) and pushes both a versioned tag and `latest` to DockerHub.

### Stage 7 — Deploy to Kubernetes
Runs on a dedicated KOPS agent. Uses Helm to upgrade/install the application chart into the `production` namespace, injecting the build-number-versioned image tag dynamically.

## Dockerfile

```dockerfile
FROM tomcat:8-jre11
RUN rm -rf /usr/local/tomcat/webapps/*
COPY target/vprofile-v2.war /usr/local/tomcat/webapps/ROOT.war
EXPOSE 8080
CMD ["catalina.sh", "run"]
```

## Prerequisites

- Jenkins with plugins: Maven Integration, Docker Pipeline, SonarQube Scanner, Kubernetes CLI
- A running Kubernetes cluster (KOPS on AWS or equivalent)
- DockerHub account and credentials configured in Jenkins
- SonarQube server configured in Jenkins global settings
- Helm 3 installed on the KOPS Jenkins agent

## How to Run

1. Fork this repository and configure it as a Jenkins Pipeline job (point to `Jenkinsfile`)
2. Set the following Jenkins credentials:
   - `dockerhub` — DockerHub username/password
   - `sonarqube` — SonarQube token
3. Configure the KOPS agent node in Jenkins with `kubectl` and `helm` access to your cluster
4. Trigger the pipeline — on success, the app will be live in the `production` namespace on port 8080

## Key Concepts Demonstrated

- **Immutable deployments**: every build produces a uniquely versioned Docker image
- **Shift-left testing**: code quality gates run before any image is built
- **GitOps-ready structure**: Helm values can be overridden per environment
- **Declarative pipeline as code**: entire CI/CD flow lives in version control
