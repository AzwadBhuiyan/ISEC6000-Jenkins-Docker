# ISEC6000 - Hardened Jenkins & Docker-in-Docker CI/CD Infrastructure

This repository contains the Infrastructure-as-Code (IaC) configuration used to provision a containerized, security-hardened Jenkins continuous integration environment. It runs a custom Jenkins LTS controller paired with an isolated Docker-in-Docker (DinD) sidecar engine to execute continuous delivery pipelines safely.

The architecture separates the build automation platform from the underlying host system, ensuring build workloads cannot modify host resources or access sensitive host processes.

---

## Architecture & Security Design

### 1. Isolated Docker-in-Docker Sidecar
Standard CI setups frequently mount the host Docker socket (`/var/run/docker.sock`) into containers to enable image builds, which grants root-equivalent control over the host daemon. 

To eliminate this privilege escalation risk:
- Jenkins runs as an unprivileged client that offloads all container build commands to a dedicated `docker:dind` sidecar service.
- The sidecar runs within an internal Docker network (`jenkins-net`), preventing exposure of Docker APIs to the public network or local host interface.

### 2. Mutual TLS (mTLS) Authentication
Communication between the Jenkins controller and the DinD daemon is encrypted and authenticated using mutual TLS over TCP port 2376 (`DOCKER_TLS_VERIFY=1`). DinD generates and verifies unique client/server certificates mounted via dedicated volumes, ensuring that only the authorized Jenkins instance can issue Docker daemon instructions.

### 3. Least Privilege & Non-Root Execution
The Jenkins controller container is built on top of `jenkins/jenkins:lts` and runs strictly under the unprivileged `jenkins` system account (`uid=1000`, `gid=1000`). Dropping root execution reduces the impact of potential remote code execution vulnerabilities or plugin compromise.

### 4. Persistent Volume Management
Named volumes isolate build data and cryptographic keys:
- `jenkins-data`: Persists jobs, build history, plugin configurations, and encrypted credential stores across container restarts.
- `jenkins-certs`: Shares client certificates between the sidecar and controller with read-only access on the client container.

---

## Repository Structure

```text
.
├── Dockerfile          # Custom Jenkins LTS image with Docker CLI client and non-root user setup
├── docker-compose.yml  # Multi-container orchestration, mTLS environment variables, and network definitions
└── README.md           # Technical documentation and deployment guidelines
```

---

## Deployment & Verification

### Prerequisites
- Docker Engine 24+ and Docker Compose v2
- Open ports: `127.0.0.1:8080` (Jenkins Web UI)

### 1. Start the Environment
Run the services in detached mode:
```bash
docker compose up -d
```

### 2. Verify Services & Network Status
Check that both services are healthy and operating on the bridge network:
```bash
docker compose ps
docker volume ls | grep isec6000
```

### 3. Verify Non-Root User & mTLS
Confirm that the Jenkins controller is running as non-root and communicating securely with the DinD daemon:
```bash
docker exec jenkins-controller whoami && docker exec jenkins-controller id
docker exec jenkins-controller docker version
```

### 4. Access Jenkins
Navigate to `http://127.0.0.1:8080` in your browser. The initial administrator setup key can be retrieved from the controller logs:
```bash
docker logs jenkins-controller 2>&1 | grep -A 2 "initialAdminPassword"
```

---

## Integrated Application Pipeline
This runner infrastructure is paired with an Express.js application pipeline configured here:
- **Application Repository**: [AzwadBhuiyan/aws-elastic-beanstalk-express-js-sample](https://github.com/AzwadBhuiyan/aws-elastic-beanstalk-express-js-sample)
- **Target Registry**: [azwadhq/isec6000-node-app](https://hub.docker.com/r/azwadhq/isec6000-node-app)
