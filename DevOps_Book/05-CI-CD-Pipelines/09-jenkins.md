# Chapter 9 — Jenkins

## Learning Objectives

By the end of this chapter you will be able to:

- Explain Jenkins' controller/agent architecture and why it scales
- Write a declarative `Jenkinsfile` with stages, parallel steps, credentials, and post-actions
- Use the Docker executor to run jobs in clean containers
- Compare Jenkins with GitHub Actions and know when to choose each

---

## 9.1 What Is Jenkins and Why It Still Matters

Jenkins is an open-source automation server first released in 2011 (originally as Hudson). It is the most widely deployed CI/CD tool in enterprise environments.

**Why Jenkins persists:**

- **Self-hosted only** — full control over infrastructure, data, and execution environment
- **1,800+ plugins** — integrations for virtually every build tool, test framework, cloud provider, and notification service
- **Language and platform agnostic** — not tied to any hosting provider's ecosystem
- **Battle-tested** — large enterprises have run Jenkins for over a decade; existing pipelines represent significant institutional investment

**Why you should learn it:**

- Many companies still run Jenkins as their primary CI/CD platform
- Migration projects (Jenkins → GitHub Actions / GitLab CI) are common, and you need to read Jenkinsfiles to migrate them
- Jenkins exposes concepts (agents, pipelines, credentials stores, shared libraries) that appear in other tools under different names

---

## 9.2 Jenkins Architecture

```
Jenkins Controller (formerly "master")
    │
    ├── stores pipeline definitions and configuration
    ├── presents the web UI and REST API
    ├── schedules jobs and distributes work to agents
    └── does NOT run build steps itself (best practice)

Jenkins Agents (formerly "slaves")
    ├── Agent 1 (Linux, general purpose)
    ├── Agent 2 (Windows, .NET builds)
    └── Agent 3 (Docker, containerized jobs)

Connection: agents connect to the controller via JNLP (port 50000)
            or via SSH from the controller to the agent
```

**Why separate controller from agent?**

- The controller is the single point of truth for configuration — it should not be loaded with build work
- Agents can be scaled horizontally; add more agents when the queue grows
- Agents can be specialized — GPU agents, Windows agents, agents with specific tools installed

---

## 9.3 Jenkinsfile — Pipeline as Code

Jenkins supports two pipeline syntaxes. Use **Declarative** for new pipelines — it is more structured and easier to read. **Scripted** (raw Groovy) is available for advanced cases.

```groovy
// Declarative Pipeline (Jenkinsfile)
pipeline {
    agent any                    // run on any available agent

    environment {
        DOCKER_REGISTRY = 'registry.mycompany.com'
        IMAGE_NAME      = 'myapp'
    }

    options {
        timeout(time: 30, unit: 'MINUTES')       // abort if pipeline takes > 30 min
        buildDiscarder(logRotator(numToKeepStr: '10'))  // keep last 10 builds
        timestamps()                             // prefix all log lines with timestamps
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm    // checks out the branch that triggered the build
            }
        }

        stage('Test') {
            steps {
                sh 'npm ci'
                sh 'npm test -- --reporters=jest-junit'
            }
            post {
                always {
                    junit 'junit.xml'
                    publishHTML([
                        reportDir:   'coverage',
                        reportFiles: 'index.html',
                        reportName:  'Coverage Report'
                    ])
                }
            }
        }

        stage('Build') {
            steps {
                script {
                    docker.build("${DOCKER_REGISTRY}/${IMAGE_NAME}:${GIT_COMMIT}")
                }
            }
        }

        stage('Push') {
            when {
                branch 'main'   // only run on the main branch
            }
            steps {
                script {
                    docker.withRegistry(
                        "https://${DOCKER_REGISTRY}",
                        'registry-credentials'      // credential ID in Jenkins store
                    ) {
                        def img = docker.image("${DOCKER_REGISTRY}/${IMAGE_NAME}:${GIT_COMMIT}")
                        img.push()
                        img.push('latest')
                    }
                }
            }
        }

        stage('Deploy to Staging') {
            when { branch 'main' }
            steps {
                sshagent(['staging-server-key']) {   // SSH credential ID
                    sh '''
                        ssh -o StrictHostKeyChecking=no deploy@staging.mycompany.com "
                            docker pull ${DOCKER_REGISTRY}/${IMAGE_NAME}:${GIT_COMMIT} &&
                            docker stop myapp || true &&
                            docker rm   myapp || true &&
                            docker run -d --name myapp -p 3000:3000 \
                              ${DOCKER_REGISTRY}/${IMAGE_NAME}:${GIT_COMMIT}
                        "
                    '''
                }
            }
        }

        stage('Deploy to Production') {
            when { branch 'main' }
            input {
                message   'Deploy to production?'
                ok        'Deploy'
                submitter 'lead-engineers'   // only users in this group can approve
            }
            steps {
                sh './deploy-prod.sh ${GIT_COMMIT}'
            }
        }
    }

    post {
        success {
            slackSend(
                color:   'good',
                message: "Build succeeded: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
            )
        }
        failure {
            slackSend(
                color:   'danger',
                message: "Build FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
            )
            emailext(
                to:      'team@mycompany.com',
                subject: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body:    "See: ${env.BUILD_URL}"
            )
        }
    }
}
```

**Key declarative blocks:**

| Block | Purpose |
|---|---|
| `agent` | Where to run: `any`, specific label, `docker { image '...' }` |
| `environment` | Set env variables for all stages |
| `options` | Pipeline-level settings (timeout, log rotation, etc.) |
| `stages` / `stage` | Logical grouping of steps |
| `steps` | The actual commands to run |
| `when` | Condition for running the stage |
| `input` | Pause and wait for human approval |
| `post` | Actions after stages complete (`always`, `success`, `failure`, `unstable`) |

---

## 9.4 Jenkins Credentials

Jenkins has a built-in credentials store. Never hardcode secrets in a Jenkinsfile — reference them by credential ID:

```groovy
// Username + password (e.g. Docker registry login)
withCredentials([
    usernamePassword(
        credentialsId: 'docker-registry',
        usernameVariable: 'REGISTRY_USER',
        passwordVariable: 'REGISTRY_PASS'
    )
]) {
    sh 'docker login -u $REGISTRY_USER -p $REGISTRY_PASS registry.mycompany.com'
}

// SSH private key
withCredentials([
    sshUserPrivateKey(
        credentialsId: 'deploy-key',
        keyFileVariable: 'SSH_KEY'
    )
]) {
    sh 'ssh -i $SSH_KEY deploy@server.com "./deploy.sh"'
}

// Secret text (API token, etc.)
withCredentials([string(credentialsId: 'api-token', variable: 'API_TOKEN')]) {
    sh 'curl -H "Authorization: Bearer $API_TOKEN" https://api.example.com/deploy'
}
```

Credentials are masked in the build log — Jenkins replaces any appearance of the secret value with `****`.

**Adding credentials in the UI:**

Dashboard → Manage Jenkins → Credentials → System → Global credentials → Add Credentials

---

## 9.5 Parallel Stages

Run independent work concurrently to reduce pipeline duration:

```groovy
stage('Verify') {
    parallel {
        stage('Unit Tests') {
            steps { sh 'npm run test:unit' }
        }
        stage('Lint') {
            steps { sh 'npm run lint' }
        }
        stage('Type Check') {
            steps { sh 'npm run type-check' }
        }
        stage('Security Scan') {
            steps { sh 'npm audit --audit-level=high' }
        }
    }
}
```

All four stages run simultaneously on available agents. The `Verify` stage is considered successful only when all parallel branches succeed.

**Parallel with different agents:**

```groovy
stage('Cross-platform Build') {
    parallel {
        stage('Linux') {
            agent { label 'linux' }
            steps { sh './build.sh linux' }
        }
        stage('Windows') {
            agent { label 'windows' }
            steps { bat 'build.bat' }
        }
    }
}
```

---

## 9.6 Docker Agent

Run the entire pipeline, or individual stages, inside a Docker container:

```groovy
// Entire pipeline in a container
pipeline {
    agent {
        docker {
            image 'node:20-alpine'
            args  '-v /tmp:/tmp'    // mount host /tmp
        }
    }
    stages {
        stage('Test') {
            steps {
                sh 'node --version'    // runs inside node:20-alpine
                sh 'npm ci && npm test'
            }
        }
    }
}

// Per-stage agent override
pipeline {
    agent none    // no default agent; each stage must declare its own

    stages {
        stage('Test') {
            agent { docker { image 'node:20-alpine' } }
            steps { sh 'npm test' }
        }
        stage('Build Image') {
            agent { docker {
                image 'docker:24'
                args  '-v /var/run/docker.sock:/var/run/docker.sock'
            }}
            steps { sh 'docker build -t myapp:latest .' }
        }
    }
}
```

The Docker agent approach replaces the need to install build tools on the Jenkins agent host directly — the agent only needs Docker installed.

---

## 9.7 Shared Libraries

Shared libraries let you extract common pipeline logic into a separate Git repository and reuse it across many Jenkinsfiles:

```groovy
// In the shared library repo: vars/deployService.groovy
def call(String serviceName, String environment) {
    echo "Deploying ${serviceName} to ${environment}"
    sh "./deploy.sh ${serviceName} ${environment}"
}
```

```groovy
// In the shared library repo: vars/runTests.groovy
def call(Map config = [:]) {
    def image  = config.image  ?: 'node:20-alpine'
    def script = config.script ?: 'npm test'
    docker.image(image).inside {
        sh script
    }
}
```

```groovy
// Usage in any project's Jenkinsfile
@Library('my-shared-lib') _

pipeline {
    agent any
    stages {
        stage('Test') {
            steps {
                runTests(image: 'node:20-alpine', script: 'npm test')
            }
        }
        stage('Deploy') {
            steps {
                deployService('myapp', 'staging')
            }
        }
    }
}
```

**Register the shared library:**

Dashboard → Manage Jenkins → Configure System → Global Pipeline Libraries → Add

Set the library name (`my-shared-lib`), Git URL, and default branch. Jenkins fetches the library automatically when a Jenkinsfile references it.

---

## 9.8 Running Jenkins with Docker

The fastest way to try Jenkins locally:

```bash
# Start Jenkins
docker run -d \
  --name jenkins \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins-data:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkins/jenkins:lts-jdk17

# Get the initial admin password
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

Then open `http://localhost:8080`, paste the password, and install the suggested plugins.

**Mounting `/var/run/docker.sock`** lets Jenkins run Docker commands on the host. This is called Docker Outside of Docker (DooD) and is simpler than full DinD.

**Jenkins Configuration as Code (JCasC)**

For reproducible Jenkins setups, use JCasC to define the entire configuration in YAML:

```yaml
# jenkins.yaml
jenkins:
  systemMessage: "Jenkins configured by JCasC"
  numExecutors: 0    # controller runs no jobs
  agentProtocols:
    - "JNLP4-connect"

credentials:
  system:
    domainCredentials:
      - credentials:
          - usernamePassword:
              scope: GLOBAL
              id: "docker-registry"
              username: "${REGISTRY_USER}"
              password: "${REGISTRY_PASS}"
```

Mount this file at `/var/jenkins_home/jenkins.yaml` and Jenkins applies it on startup.

---

## 9.9 Jenkins vs GitHub Actions

| Feature | Jenkins | GitHub Actions |
|---|---|---|
| Hosting | Self-hosted only | GitHub cloud (free tier available) |
| Config language | Groovy (Declarative DSL) | YAML |
| Plugins | 1,800+ | 10,000+ marketplace actions |
| Flexibility | Maximum — custom everything | High within GitHub ecosystem |
| Infrastructure maintenance | High — you own the servers | None for cloud runners |
| Cost | Infrastructure cost only | Free for public repos; minutes for private |
| Learning curve | Steeper (Groovy, plugin model) | Moderate |
| Built-in registry | No (use Nexus, Harbor, etc.) | GHCR available |
| Kubernetes support | Jenkins X, JCasC, K8s plugin | Self-hosted runner in a pod |
| Multi-branch pipelines | Built-in | Workflow per branch automatically |
| Approval gates | `input` block | `environment: required reviewers` |

**Choose Jenkins when:**

- Your organization is self-hosted and cannot use SaaS CI
- You have existing Jenkins pipelines that are not worth migrating
- You need deep customization that no hosted CI tool provides
- Compliance requires that build artifacts never leave your infrastructure

**Choose GitHub Actions when:**

- Your code is already on GitHub
- You want minimal infrastructure maintenance
- You want access to the large ecosystem of pre-built actions

---

## Summary

- Jenkins separates the controller (orchestration, UI) from agents (execution) — scale by adding agents
- Declarative `Jenkinsfile` syntax is the recommended approach; use `scripted` only when you hit its limits
- The credentials store keeps secrets out of your code; reference credentials by ID inside `withCredentials` blocks
- `parallel` stages reduce pipeline duration by running independent work concurrently
- The Docker agent pattern gives clean, reproducible environments without installing tools on agent hosts
- Shared libraries extract common logic into a reusable Git repository — the equivalent of reusable GitHub Actions
- Jenkins requires more operational effort than hosted CI tools but provides maximum flexibility and control

---

## Knowledge Check

1. What is the difference between the Jenkins controller and a Jenkins agent? Why should the controller not execute build steps?
2. In the declarative pipeline syntax, what is the difference between `when` and `input`? When would you use each?
3. Why is it important to use `withCredentials` rather than setting secret values directly in `environment {}` or in `sh` commands?
4. Your pipeline has four independent verification steps (unit tests, lint, type check, security scan). How do you run them concurrently and ensure the pipeline fails if any one of them fails?
5. What is a Jenkins shared library and what problem does it solve?

---

## Hands-on Exercise

**Goal:** Run Jenkins locally with Docker, create a pipeline that clones a repo, runs tests, and builds a Docker image.

**Steps:**

1. Start Jenkins using the Docker command in section 9.8. Complete the initial setup and install suggested plugins.
2. Install the **Docker Pipeline** plugin (Manage Jenkins → Plugins → Available → search "Docker Pipeline").
3. Create a new item → **Pipeline** and paste the following `Jenkinsfile` into the pipeline script box:

```groovy
pipeline {
    agent {
        docker { image 'node:20-alpine' }
    }
    stages {
        stage('Checkout') {
            steps { checkout scm }
        }
        stage('Install') {
            steps { sh 'npm ci' }
        }
        stage('Test') {
            steps { sh 'npm test' }
        }
        stage('Build Summary') {
            steps {
                echo "Tests passed for commit: ${GIT_COMMIT}"
                echo "Build number: ${env.BUILD_NUMBER}"
            }
        }
    }
    post {
        success { echo 'Pipeline succeeded.' }
        failure { echo 'Pipeline failed.' }
    }
}
```

4. Add a Docker Hub or registry credential to the Jenkins credentials store (Dashboard → Manage Jenkins → Credentials).
5. Extend the pipeline with a `Build Docker Image` stage that uses `docker.withRegistry` to push the image to your registry.
6. Trigger the pipeline by clicking **Build Now** and examine the stage view to understand how Jenkins visualizes the pipeline.

**Stretch goal:** Configure a multibranch pipeline pointing at a real GitHub repository and observe how Jenkins automatically creates a pipeline for each branch that has a `Jenkinsfile`.

---

<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="08-gitlab-ci.md">← Previous: GitLab CI</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="10-secrets-management.md">Next: Secrets Management →</a>
</div>
