# Jenkins — CI/CD Deep Dive (6 YOE)

---

## What is Jenkins?

Jenkins is an open-source **automation server** for Continuous Integration (CI) and Continuous Delivery (CD). It automates the build, test, and deploy pipeline triggered by code changes.

```
Developer pushes code → Jenkins detects (webhook/poll) → runs pipeline:
  Checkout → Build → Test → Docker Build → Push → Deploy → Notify
```

---

## Jenkins Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                      JENKINS MASTER (Controller)                      │
│                                                                       │
│  ┌─────────┐  ┌──────────┐  ┌───────────┐  ┌────────────────────┐  │
│  │  Web UI │  │   REST   │  │  Plugin   │  │   Job Scheduler    │  │
│  │  (8080) │  │   API    │  │  Manager  │  │   & Queue          │  │
│  └─────────┘  └──────────┘  └───────────┘  └────────────────────┘  │
│                                                                       │
│  Does NOT run builds (security/performance best practice)             │
└──────────────────────────────────────────────────────────────────────┘
         │ distributes work via JNLP / SSH
         ▼
┌────────────────────────────────────────────────────────────────────┐
│                         JENKINS AGENTS (Workers)                    │
│                                                                     │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────────────────────┐   │
│  │  Linux Agent │  │ Linux Agent │  │  Kubernetes Pod Agent    │   │
│  │  (on-prem)  │  │  (GCE VM)   │  │  (dynamic, ephemeral)    │   │
│  │  Labels:    │  │  Labels:    │  │  Labels: k8s-agent        │   │
│  │  linux,java │  │  docker     │  │  Spins up for job, dies   │   │
│  └─────────────┘  └─────────────┘  └──────────────────────────┘   │
└────────────────────────────────────────────────────────────────────┘
```

**Kubernetes Plugin**: Jenkins dynamically creates pods in K8s cluster as agents. Each build gets a fresh pod → no state contamination, infinite scale.

---

## Jenkins Pipeline — Declarative vs Scripted

### Declarative Pipeline (Recommended, structured)

```groovy
// Jenkinsfile (stored in repo root)
pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: build
    image: golang:1.21-alpine
    command: ['sleep', 'infinity']
  - name: docker
    image: docker:24-dind
    securityContext:
      privileged: true
    volumeMounts:
    - name: docker-sock
      mountPath: /var/run/docker.sock
  volumes:
  - name: docker-sock
    hostPath:
      path: /var/run/docker.sock
"""
        }
    }

    environment {
        APP_NAME    = 'myapp'
        GCR_REPO    = 'gcr.io/my-project/myapp'
        IMAGE_TAG   = "${env.GIT_COMMIT[0..7]}"
        PROD_NAMESPACE = 'production'
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '20'))
        disableConcurrentBuilds()
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Test') {
            steps {
                container('build') {
                    sh '''
                        go mod download
                        go test ./... -v -race -coverprofile=coverage.out
                        go vet ./...
                    '''
                }
            }
            post {
                always {
                    junit 'test-results/**/*.xml'
                    publishCoverage adapters: [coberturaAdapter('coverage.xml')]
                }
            }
        }

        stage('Docker Build') {
            steps {
                container('docker') {
                    sh "docker build -t ${GCR_REPO}:${IMAGE_TAG} ."
                    sh "docker tag ${GCR_REPO}:${IMAGE_TAG} ${GCR_REPO}:latest"
                }
            }
        }

        stage('Security Scan') {
            steps {
                container('docker') {
                    sh "trivy image --exit-code 1 --severity HIGH,CRITICAL ${GCR_REPO}:${IMAGE_TAG}"
                }
            }
        }

        stage('Push to GCR') {
            steps {
                container('docker') {
                    withCredentials([file(credentialsId: 'gcr-sa-key', variable: 'GCR_KEY')]) {
                        sh """
                            cat ${GCR_KEY} | docker login -u _json_key --password-stdin https://gcr.io
                            docker push ${GCR_REPO}:${IMAGE_TAG}
                            docker push ${GCR_REPO}:latest
                        """
                    }
                }
            }
        }

        stage('Deploy to Staging') {
            when {
                branch 'main'
            }
            steps {
                sh """
                    kubectl set image deployment/${APP_NAME} \
                        app=${GCR_REPO}:${IMAGE_TAG} \
                        -n staging
                    kubectl rollout status deployment/${APP_NAME} -n staging --timeout=5m
                """
            }
        }

        stage('Integration Tests') {
            when {
                branch 'main'
            }
            steps {
                sh "./scripts/integration-tests.sh https://staging.example.com"
            }
        }

        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            input {
                message "Deploy to production?"
                ok "Yes, deploy!"
                submitter "senior-devs,sre-team"
            }
            steps {
                sh """
                    kubectl set image deployment/${APP_NAME} \
                        app=${GCR_REPO}:${IMAGE_TAG} \
                        -n ${PROD_NAMESPACE}
                    kubectl rollout status deployment/${APP_NAME} \
                        -n ${PROD_NAMESPACE} --timeout=10m
                """
            }
        }
    }

    post {
        success {
            slackSend(
                channel: '#deployments',
                color: 'good',
                message: "SUCCESS: ${APP_NAME} ${IMAGE_TAG} deployed to prod"
            )
        }
        failure {
            slackSend(
                channel: '#alerts',
                color: 'danger',
                message: "FAILED: ${APP_NAME} pipeline - ${env.BUILD_URL}"
            )
            // Auto-rollback on deploy failure
            sh "kubectl rollout undo deployment/${APP_NAME} -n ${PROD_NAMESPACE}"
        }
        always {
            cleanWs()  // Clean workspace after build
        }
    }
}
```

---

## CI/CD Pipeline Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     JENKINS PIPELINE STAGES                              │
│                                                                          │
│  GitHub                                                                  │
│  Push/PR ──► [Webhook] ──► Jenkins                                       │
│                              │                                           │
│              ┌───────────────┼───────────────────────────────┐           │
│              │               ▼                               │           │
│              │    ┌─────────────────┐                        │           │
│              │    │   1. Checkout   │ git clone              │           │
│              │    └────────┬────────┘                        │           │
│              │             ▼                                 │           │
│              │    ┌─────────────────┐                        │           │
│              │    │   2. Lint/Test  │ unit tests, vet        │           │
│              │    └────────┬────────┘                        │           │
│              │             ▼                                 │           │
│              │    ┌─────────────────┐                        │           │
│              │    │   3. Build      │ docker build           │           │
│              │    └────────┬────────┘                        │           │
│              │             ▼                                 │           │
│              │    ┌─────────────────┐                        │           │
│              │    │ 4. Scan (trivy) │ CVE check              │           │
│              │    └────────┬────────┘                        │           │
│              │             ▼                                 │           │
│              │    ┌─────────────────┐                        │           │
│              │    │   5. Push GCR   │ tag + push             │           │
│              │    └────────┬────────┘                        │           │
│  PR builds:  │             ▼           (main branch only)    │           │
│  stop here   │    ┌─────────────────┐                        │           │
│              │    │ 6. Deploy Stage │ kubectl set image      │           │
│              │    └────────┬────────┘                        │           │
│              │             ▼                                 │           │
│              │    ┌─────────────────┐                        │           │
│              │    │ 7. Integration  │ smoke tests            │           │
│              │    │    Tests        │                        │           │
│              │    └────────┬────────┘                        │           │
│              │             ▼                                 │           │
│              │    ┌─────────────────┐                        │           │
│              │    │ 8. Manual Gate  │ human approval         │           │
│              │    └────────┬────────┘                        │           │
│              │             ▼                                 │           │
│              │    ┌─────────────────┐                        │           │
│              │    │ 9. Deploy Prod  │ rolling update         │           │
│              │    └────────┬────────┘                        │           │
│              │             ▼                                 │           │
│              │    ┌─────────────────┐                        │           │
│              │    │ 10. Notify      │ Slack/PagerDuty        │           │
│              │    └─────────────────┘                        │           │
│              └───────────────────────────────────────────────┘           │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Jenkins Plugins (Essential)

| Plugin | Purpose |
|--------|---------|
| Pipeline | Core pipeline support |
| Kubernetes | Dynamic pod agents in K8s |
| Git / GitHub | Source code integration, webhooks |
| Credentials Binding | Inject secrets into builds |
| Docker Pipeline | Docker build/push in pipeline |
| Slack Notification | Notify #channels |
| Blue Ocean | Modern pipeline UI |
| Parameterized Trigger | Trigger downstream jobs |
| Shared Libraries | Reuse pipeline code across projects |

---

## Jenkins Shared Libraries

Centralize common pipeline logic across all repos.

```
jenkins-shared-lib/
├── vars/
│   ├── dockerBuild.groovy     # call as: dockerBuild(image: 'myapp', tag: '1.0')
│   ├── deployToGKE.groovy
│   └── sendSlackAlert.groovy
├── src/
│   └── org/company/Utils.groovy
└── resources/
    └── podTemplates/build-pod.yaml
```

```groovy
// vars/deployToGKE.groovy
def call(Map config) {
    sh """
        kubectl set image deployment/${config.appName} \
            app=${config.image}:${config.tag} \
            -n ${config.namespace}
        kubectl rollout status deployment/${config.appName} \
            -n ${config.namespace} --timeout=10m
    """
}

// In Jenkinsfile
@Library('jenkins-shared-lib') _

pipeline {
    stages {
        stage('Deploy') {
            steps {
                deployToGKE(appName: 'myapp', image: 'gcr.io/proj/myapp', tag: '1.0.0', namespace: 'production')
            }
        }
    }
}
```

---

## Credentials Management

```groovy
// Environment variable
withCredentials([string(credentialsId: 'api-key', variable: 'API_KEY')]) {
    sh 'curl -H "Authorization: ${API_KEY}" ...'
}

// File credential
withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
    sh 'kubectl get pods'
}

// Username/password
withCredentials([usernamePassword(credentialsId: 'db-creds',
                                  usernameVariable: 'USER',
                                  passwordVariable: 'PASS')]) {
    sh 'psql -U ${USER} -W ${PASS}'
}
```

**Best Practice**: Use Jenkins credentials store (backed by GCP Secret Manager via plugin) instead of hardcoded secrets in Jenkinsfiles.

---

## Jenkins High Availability

```
Standard Jenkins: Single master → SPOF
HA Options:
  - CloudBees Jenkins (commercial HA)
  - Jenkins on K8s with persistent volume (semi-HA)
  - GitHub Actions / Cloud Build as alternative (true HA SaaS)

Backup strategy:
  - Backup $JENKINS_HOME (jobs, config, credentials)
  - Store Jenkinsfiles in repos (not in Jenkins)
  - Use Job DSL / JCasC (Jenkins Configuration as Code) for repeatable setup
```

---

## Jenkins Configuration as Code (JCasC)

```yaml
# jenkins.yaml — define entire Jenkins config declaratively
jenkins:
  systemMessage: "Managed by JCasC"
  numExecutors: 0  # master runs no jobs
  mode: EXCLUSIVE

  clouds:
  - kubernetes:
      name: kubernetes
      serverUrl: https://kubernetes.default
      namespace: jenkins
      jenkinsUrl: http://jenkins:8080
      jenkinsTunnel: jenkins-agent:50000
      podTemplates:
      - name: default
        containers:
        - name: jnlp
          image: jenkins/inbound-agent:latest

credentials:
  system:
    domainCredentials:
    - credentials:
      - string:
          id: slack-token
          secret: ${SLACK_TOKEN}   # from env/secret
```

---

## Interview Questions — Jenkins (6 YOE Level)

**Q: What's the difference between Declarative and Scripted pipelines?**
> **Declarative**: Structured, predefined syntax, easier to read, built-in validation, `pipeline { }` block. Recommended. **Scripted**: Full Groovy DSL, `node { }` block, more flexible but harder to maintain. Use Declarative for most cases; fall back to `script { }` blocks inside Declarative when you need scripted flexibility.

**Q: How do you handle secrets in Jenkins pipelines?**
> Store in Jenkins Credentials Store (or backed by HashiCorp Vault/GCP Secret Manager). Inject via `withCredentials()` block. Never echo secrets in logs (`set +x`). Mask credentials via Jenkins built-in masking. Never store secrets in Jenkinsfiles.

**Q: How do you achieve zero-downtime deployments in Jenkins?**
> Deploy to K8s using rolling updates: `kubectl set image` + `kubectl rollout status`. Ensure readiness probes are configured. Use `maxUnavailable: 0` in Deployment strategy. Add automatic rollback in `post { failure }` block with `kubectl rollout undo`.

**Q: How do you scale Jenkins?**
> Use distributed builds with many agents. Use Kubernetes plugin for ephemeral pod agents (infinite scale). Use node labels to route specific builds to specific agents. Use parallel stages in pipeline. For true scale, consider migrating to GitHub Actions or Cloud Build (serverless CI).

**Q: What is a Multibranch Pipeline?**
> Jenkins automatically discovers all branches/PRs in a repo and creates pipeline jobs for each. Uses Jenkinsfile from each branch. Ideal for GitFlow — `main`, `develop`, feature branches each get their own build with appropriate stages (PRs only build+test, main builds+deploys).
