# CI/CD Pipeline: Docker + Kubernetes + Jenkins

An end-to-end CI/CD pipeline that builds a Java web application, runs automated quality and security gates, containerizes it with Docker, and deploys to a Kubernetes cluster using Helm charts.

## Architecture

```mermaid
flowchart LR
    A([Developer Push]) --> B

    subgraph Jenkins Pipeline
        B[Build\nMaven WAR] --> C[Unit Tests]
        C --> D[Integration\nTests]
        D --> E[Checkstyle\nCode Quality]
        E --> F[Docker\nBuild Image]
        F --> G{Trivy\nSecurity Scan}
        G -->|HIGH/CRITICAL CVEs| X([Abort Pipeline])
        G -->|Clean| H[Push to\nDockerHub]
        H --> I[Helm Deploy\nvia KOPS]
    end

    I --> J([Kubernetes\nProduction])
    Jenkins Pipeline -->|Success| K([Slack\nNotification ✅])
    Jenkins Pipeline -->|Failure| L([Slack\nNotification ❌])
```

## Tech Stack

| Layer | Technology |
|---|---|
| Application | Java, Spring MVC, Maven |
| Containerization | Docker, Tomcat 8 / JRE 11 |
| CI/CD | Jenkins (declarative pipeline) |
| Security Scanning | Trivy (HIGH/CRITICAL CVE gate) |
| Container Registry | DockerHub |
| Orchestration | Kubernetes (KOPS) |
| Packaging | Helm |
| Code Quality | Checkstyle |
| Alerting | Slack |

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
**Checkstyle** enforces coding standards and generates a report — failing the build on violations.

### Stage 5 — Docker Build
Builds a Docker image tagged with the Jenkins build number, producing an immutable, versioned artifact.

### Stage 6 — Trivy Security Scan
Scans the freshly built Docker image for known CVEs using [Trivy](https://github.com/aquasecurity/trivy). The pipeline **aborts immediately** if any HIGH or CRITICAL vulnerabilities are found — the image is never pushed to the registry unless it is clean. This gates security at the build stage rather than discovering issues post-deployment.

### Stage 7 — Push to DockerHub
Pushes both a versioned tag (`V$BUILD_NUMBER`) and `latest` to DockerHub only after the security scan passes.

### Stage 8 — Deploy to Kubernetes
Runs on a dedicated KOPS agent. Uses Helm to upgrade/install the application chart into the `production` namespace, injecting the build-number-versioned image tag dynamically.

### Post — Slack Notifications
Sends a success or failure message to `#devops-notifications` after every pipeline run, including the build number, deployed image tag, duration, and a direct link to the Jenkins build log.

## Dockerfile

```dockerfile
FROM tomcat:8-jre11
RUN rm -rf /usr/local/tomcat/webapps/*
COPY target/vprofile-v2.war /usr/local/tomcat/webapps/ROOT.war
EXPOSE 8080
CMD ["catalina.sh", "run"]
```

## Prerequisites

- Jenkins with plugins: Maven Integration, Docker Pipeline, Slack Notification, Kubernetes CLI
- A running Kubernetes cluster (KOPS on AWS or equivalent)
- DockerHub account and credentials configured in Jenkins (`dockerhub`)
- Trivy installed on the Jenkins agent (`apt install trivy` or via the [official install script](https://aquasecurity.github.io/trivy/latest/getting-started/installation/))
- Helm 3 installed on the KOPS Jenkins agent
- Slack app with an incoming webhook, configured in Jenkins global settings

## How to Run

1. Fork this repository and configure it as a Jenkins Pipeline job (point to `Jenkinsfile`)
2. Set the following Jenkins credentials:
   - `dockerhub` — DockerHub username/password
3. Configure Slack in Jenkins → Manage Jenkins → Slack, set the workspace and default channel
4. Configure the KOPS agent node in Jenkins with `kubectl` and `helm` access to your cluster
5. Trigger the pipeline — on success, the app is live in the `production` namespace and a Slack message confirms the deployed image tag

## Key Concepts Demonstrated

- **Shift-left security**: Trivy scans the image before it ever reaches the registry
- **Immutable deployments**: every build produces a uniquely versioned Docker image
- **Declarative pipeline as code**: entire CI/CD flow lives in version control
- **GitOps-ready structure**: Helm values can be overridden per environment
- **Closed-loop alerting**: Slack notifications close the feedback loop to the team on every build outcome
