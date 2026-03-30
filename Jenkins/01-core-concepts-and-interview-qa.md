# Jenkins - Core Concepts, Interview Q&A & Scenarios
> Senior DevOps/SRE Level | CI/CD Expert

---

## CORE CONCEPTS

### What is Jenkins?
Jenkins is an open-source automation server used for CI/CD pipelines. Written in Java, plugin-based architecture with 1800+ plugins.

### Jenkins Architecture
- **Master (Controller)**: Manages agents, schedules jobs, stores configuration and history
- **Agent (Worker)**: Executes build steps. Can be permanent or ephemeral (Kubernetes, Docker).
- **Executor**: Thread on agent that runs a build

### Pipeline Types
1. **Freestyle Project**: GUI-configured, simple jobs
2. **Scripted Pipeline**: Groovy DSL, uses node{} blocks. Flexible but complex.
3. **Declarative Pipeline**: Structured syntax, easier to read. Recommended.

---

## DECLARATIVE PIPELINE

### Complete Pipeline Example
```groovy
pipeline {
  agent {
    kubernetes {
      yaml '''
        apiVersion: v1
        kind: Pod
        spec:
          containers:
          - name: docker
            image: docker:24-dind
            securityContext:
              privileged: true
          - name: tools
            image: alpine/helm:3.12
      '''
    }
  }

  environment {
    REGISTRY = 'gcr.io/myproject'
    IMAGE_NAME = 'myapp'
    IMAGE_TAG = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
  }

  options {
    timeout(time: 30, unit: 'MINUTES')
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '10'))
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Test') {
      steps {
        sh 'go test ./...'
      }
      post {
        always {
          junit 'test-results/*.xml'
        }
      }
    }

    stage('Build & Push') {
      steps {
        container('docker') {
          sh '''
            docker build -t ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} .
            docker push ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
          '''
        }
      }
    }

    stage('Deploy to Staging') {
      when {
        branch 'main'
      }
      steps {
        container('tools') {
          sh '''
            helm upgrade --install myapp ./charts/myapp \
              --namespace staging \
              --set image.tag=${IMAGE_TAG}
          '''
        }
      }
    }

    stage('Deploy to Production') {
      when {
        tag pattern: 'v*', comparator: 'GLOB'
      }
      input {
        message 'Deploy to production?'
        ok 'Yes, deploy!'
        submitter 'sre-team'
      }
      steps {
        container('tools') {
          sh '''
            helm upgrade --install myapp ./charts/myapp \
              --namespace production \
              --set image.tag=${IMAGE_TAG}
          '''
        }
      }
    }
  }

  post {
    success {
      slackSend channel: '#deployments', color: 'good',
                message: "${env.JOB_NAME} - ${env.BUILD_NUMBER} deployed ${IMAGE_TAG}"
    }
    failure {
      slackSend channel: '#alerts', color: 'danger',
                message: "${env.JOB_NAME} - ${env.BUILD_NUMBER} FAILED"
    }
    always {
      cleanWs()  // Clean workspace
    }
  }
}
```

---

## JENKINS ON KUBERNETES

### Dynamic Agents (Jenkins Kubernetes Plugin)
```groovy
// Each build spins up a fresh K8s pod as agent
// Deleted after build completes
// Advantages: Clean environment, auto-scaling, no agent maintenance

agent {
  kubernetes {
    label 'build-pod'
    defaultContainer 'jnlp'
    yaml '''
      apiVersion: v1
      kind: Pod
      metadata:
        labels:
          app: jenkins-agent
      spec:
        serviceAccountName: jenkins-sa
        containers:
        - name: maven
          image: maven:3.9-eclipse-temurin-17
          command: [cat]
          tty: true
    '''
  }
}
```

---

## INTERVIEW QUESTIONS & ANSWERS

### Q1: What is the difference between Declarative and Scripted Pipeline?

| Aspect | Declarative | Scripted |
|---|---|---|
| Syntax | Structured (pipeline{}) | Groovy DSL (node{}) |
| Learning curve | Easier | Steeper |
| Flexibility | Less flexible | Very flexible |
| Error messages | Better | Less informative |
| Recommendation | **Preferred** | Legacy/complex cases |

### Q2: How do you pass credentials securely in Jenkins?
```groovy
// Store in Jenkins Credentials store, never in Jenkinsfile
withCredentials([
  string(credentialsId: 'docker-registry-token', variable: 'REGISTRY_TOKEN'),
  usernamePassword(credentialsId: 'db-creds',
                   usernameVariable: 'DB_USER',
                   passwordVariable: 'DB_PASS'),
  sshUserPrivateKey(credentialsId: 'deploy-key',
                    keyFileVariable: 'SSH_KEY')
]) {
  sh 'docker login -u $DB_USER -p $REGISTRY_TOKEN'
}
// Credentials are masked in logs automatically
```

### Q3: What is a shared library and why use it?
Shared Libraries centralize reusable pipeline code across teams.
Stored in a separate Git repo, imported in Jenkinsfiles.

```groovy
// In Jenkinsfile
@Library('my-shared-lib@main') _

// Use shared step
buildAndPushDocker(image: 'myapp', tag: env.GIT_COMMIT)
deployToKubernetes(namespace: 'production', chart: './charts/myapp')
```

**Benefits**: DRY (Don't Repeat Yourself), consistent standards across teams, version-controlled.

### Q4: How do you implement approval gates in Jenkins?
```groovy
stage('Deploy to Production') {
  input {
    message 'Approve production deployment?'
    ok 'Deploy'
    submitter 'sre-lead,ops-team'  // Only these users can approve
    parameters {
      choice(name: 'REASON', choices: ['Regular release', 'Hotfix'], description: 'Reason')
    }
  }
  steps {
    // Deploy only if approved
  }
}
```

### Q5: Jenkins vs GitHub Actions vs ArgoCD — when to use what?

| Tool | Best For |
|---|---|
| **Jenkins** | Complex pipelines, enterprise, existing infra, Java/Maven builds |
| **GitHub Actions** | GitHub-hosted projects, simple to medium pipelines, serverless CI |
| **ArgoCD** | CD (deployment) only, GitOps, Kubernetes deployments |

**Pattern**: GitHub Actions for CI (build, test, push image) + ArgoCD for CD (deploy to K8s).
Jenkins still common for complex enterprise builds with many integrations.

### Q6: How do you handle Jenkins at scale?
1. **Dynamic agents**: Kubernetes plugin — no permanent agents, auto-scaling
2. **Pipeline as code**: All pipelines in Git (Jenkinsfiles)
3. **Shared libraries**: DRY, standardized pipeline patterns
4. **Job DSL / JobDSL Plugin**: Auto-create/manage jobs from code
5. **Jenkins Configuration as Code (JCasC)**: Jenkins master config in YAML
6. **Distributed builds**: Multiple agents in different regions
7. **Pipeline caching**: Cache Maven/npm dependencies between builds

### Q7: How do you secure Jenkins?
1. **Matrix-based security**: Role-based access for users/groups
2. **Credentials store**: All secrets stored in Jenkins credentials (not env vars in Jenkinsfile)
3. **Script Security Plugin**: Sandbox for Groovy code
4. **CSRF protection**: Enable in security settings
5. **No admin access for build agents**: Service accounts with minimal permissions
6. **HTTPS only**: TLS termination at Nginx/LB
7. **Audit logs**: Track all user actions

### Q8: Pipeline triggers
```groovy
triggers {
  // Trigger on Git push (webhook)
  githubPush()

  // Cron schedule
  cron('H 2 * * 1-5')  // Weekdays at 2 AM

  // Poll SCM (less preferred — use webhooks)
  pollSCM('H/15 * * * *')  // Every 15 minutes

  // Upstream job trigger
  upstream(upstreamProjects: 'build-job', threshold: hudson.model.Result.SUCCESS)
}
```

---

## REAL-WORLD SCENARIOS

### Scenario: Build fails intermittently — hard to debug
```groovy
// Add retry for flaky network operations
retry(3) {
  sh 'docker pull myimage:base'
}

// Archive artifacts for failed builds
post {
  failure {
    archiveArtifacts artifacts: 'logs/**', allowEmptyArchive: true
  }
}

// Increase verbosity
sh 'set -x; ./run-tests.sh'
```

### Scenario: Pipeline runs too long
```groovy
// Run stages in parallel
stage('Test') {
  parallel {
    stage('Unit Tests') {
      steps { sh 'make unit-test' }
    }
    stage('Integration Tests') {
      steps { sh 'make integration-test' }
    }
    stage('Security Scan') {
      steps { sh 'trivy image myapp:latest' }
    }
  }
}
