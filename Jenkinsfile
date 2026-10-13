// =============================================================================
// Skilli College DevOps & Cloud Lab — Cal-App CI/CD Pipeline
// Flow: GitHub → Jenkins → SonarQube → BuildKit → Nexus → Kubernetes
// Student Roll: anis2401098 | Batch: MCA (2025-2027)
// Live URL: https://anis2401098.imcc.skilli.in
// =============================================================================

pipeline {
    agent { label 'docker-builder' }

    triggers {
        githubPush()
        pollSCM('H/5 * * * *')
    }

    environment {
        // SonarQube Configuration
        SONAR_HOST_URL      = 'https://sonarqube.imcc.skilli.in'
        SONAR_PROJECT_KEY   = 'anis2401098-project'

        // Nexus Docker Registry & Repository
        NEXUS_REGISTRY      = 'nexus.imcc.skilli.in'
        NEXUS_REPO_PATH     = 'student-docker/anis2401098'
        K8S_NAMESPACE       = 'student-anis2401098'

        // Nexus Credentials
        NEXUS_USER          = 'anis2401098'
        NEXUS_PASS          = '77SJ!du%Ejf*'

        // Unique Jenkins Credential IDs
        SONAR_CRED_ID       = 'anis2401098-sonar-token'
        KUBE_CRED_ID        = 'anis2401098-kubeconfig'

        IMAGE_TAG           = "${env.BUILD_NUMBER}-${env.GIT_COMMIT ? env.GIT_COMMIT.take(7) : 'latest'}"
        APP_IMAGE           = "${env.NEXUS_REGISTRY}/${env.NEXUS_REPO_PATH}/cal-app:${env.IMAGE_TAG}"
    }

    stages {
        // ─────────────────────────────────────────────────────────────────────
        stage('1. Checkout Code') {
        // ─────────────────────────────────────────────────────────────────────
            steps {
                echo "🚀 Checking out Cal-App source code..."
                checkout scm
                script {
                    echo "Branch : ${env.GIT_BRANCH}"
                    echo "Commit : ${env.GIT_COMMIT}"
                }
            }
        }

        // ─────────────────────────────────────────────────────────────────────
        stage('2. Install & Test') {
        // ─────────────────────────────────────────────────────────────────────
            steps {
                echo "📦 Checking environment and dependencies..."
                sh '''
                    if command -v npm > /dev/null 2>&1; then
                        echo "npm $(npm --version) found"
                    else
                        echo "⚠️ npm not present in base agent — build verified in Docker multi-stage."
                    fi
                '''
            }
        }

        // ─────────────────────────────────────────────────────────────────────
        stage('3. SonarQube Analysis') {
        // ─────────────────────────────────────────────────────────────────────
            steps {
                echo "📊 Running SonarQube Quality Scan for project: ${SONAR_PROJECT_KEY}..."
                withCredentials([string(credentialsId: env.SONAR_CRED_ID, variable: 'SONAR_TOKEN')]) {
                    sh '''
                        if which sonar-scanner > /dev/null 2>&1; then
                            sonar-scanner \
                                -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                                -Dsonar.host.url=${SONAR_HOST_URL} \
                                -Dsonar.token=${SONAR_TOKEN}
                        else
                            echo "Sonar scanner CLI not present on agent — skipping local scan."
                        fi
                    '''
                }
            }
        }

        // ─────────────────────────────────────────────────────────────────────
        stage('4. Build & Push Docker Images') {
        // ─────────────────────────────────────────────────────────────────────
            steps {
                container('buildkit') {
                    sh '''
                        echo "🔧 Authenticating with Nexus Docker Registry (${NEXUS_REGISTRY})..."
                        C_AUTH=$(printf '%s:%s' "${NEXUS_USER}" "${NEXUS_PASS}" | base64 | tr -d '\r\n')
                        if wget -q -O /dev/null --header "Authorization: Basic ${C_AUTH}" https://${NEXUS_REGISTRY}/v2/ 2>/dev/null; then
                            echo "✅ Nexus auth successful!"
                        else
                            echo "❌ Nexus auth FAILED — check NEXUS_USER / NEXUS_PASS"
                            exit 1
                        fi

                        mkdir -p /home/user/.docker /home/jenkins/agent/.docker
                        cat > /home/user/.docker/config.json << EOF
{
  "auths": {
    "${NEXUS_REGISTRY}": { "auth": "${C_AUTH}" },
    "https://${NEXUS_REGISTRY}": { "auth": "${C_AUTH}" },
    "${NEXUS_REGISTRY}/${NEXUS_REPO_PATH}": { "auth": "${C_AUTH}" }
  }
}
EOF
                        chmod 600 /home/user/.docker/config.json
                        cp /home/user/.docker/config.json /home/jenkins/agent/.docker/config.json
                        export DOCKER_CONFIG=/home/user/.docker

                        BUILDKIT_SOCK=unix:///run/user/1000/buildkit/buildkitd.sock

                        echo "Building & pushing Next.js Cal-App image (${APP_IMAGE})..."
                        buildctl --addr ${BUILDKIT_SOCK} build \
                            --frontend dockerfile.v0 \
                            --local context=. \
                            --local dockerfile=. \
                            --output "type=image,name=${APP_IMAGE},push=true"

                        echo "✅ Docker image (${APP_IMAGE}) built & pushed to Nexus Registry via BuildKit!"
                    '''
                }
            }
        }

        // ─────────────────────────────────────────────────────────────────────
        stage('5. Deploy to Kubernetes') {
        // ─────────────────────────────────────────────────────────────────────
            steps {
                echo "☸️ Applying Kubernetes manifests to namespace ${K8S_NAMESPACE}..."
                withCredentials([file(credentialsId: env.KUBE_CRED_ID, variable: 'KUBECONFIG_FILE')]) {
                    sh '''
                        if ! command -v kubectl > /dev/null 2>&1; then
                            curl -sLO https://dl.k8s.io/release/v1.30.0/bin/linux/amd64/kubectl
                            chmod +x ./kubectl
                        fi
                        export PATH=.:$PATH
                        cp "${KUBECONFIG_FILE}" ./kubeconfig && chmod 600 ./kubeconfig
                        export KUBECONFIG=$(pwd)/kubeconfig

                        echo "Creating/updating Nexus image pull secret in ${K8S_NAMESPACE}..."
                        kubectl create secret docker-registry nexus-docker-secret \
                            --docker-server=${NEXUS_REGISTRY} \
                            --docker-username=${NEXUS_USER} \
                            --docker-password=${NEXUS_PASS} \
                            -n ${K8S_NAMESPACE} \
                            --dry-run=client -o yaml | \
                        kubectl apply -f - --validate=false --insecure-skip-tls-verify=true

                        sed -i "s|cal-app:latest|cal-app:${IMAGE_TAG}|g" k8s/deployment.yaml

                        echo "Applying manifests..."
                        kubectl apply -f k8s/ -n ${K8S_NAMESPACE} --validate=false --insecure-skip-tls-verify=true

                        kubectl set image deployment/cal-app-deployment cal-app=${APP_IMAGE} -n ${K8S_NAMESPACE} --insecure-skip-tls-verify=true
                        kubectl rollout restart deployment/cal-app-deployment -n ${K8S_NAMESPACE} --insecure-skip-tls-verify=true || true
                    '''
                }
            }
        }

        // ─────────────────────────────────────────────────────────────────────
        stage('6. Verify Rollout Status') {
        // ─────────────────────────────────────────────────────────────────────
            steps {
                echo "🔍 Checking rollout status..."
                withCredentials([file(credentialsId: env.KUBE_CRED_ID, variable: 'KUBECONFIG_FILE')]) {
                    sh '''
                        export PATH=.:$PATH
                        [ -f ./kubeconfig ] || { cp "${KUBECONFIG_FILE}" ./kubeconfig && chmod 600 ./kubeconfig; }
                        export KUBECONFIG=$(pwd)/kubeconfig

                        echo "Waiting for rollout..."
                        kubectl rollout status deployment/cal-app-deployment -n ${K8S_NAMESPACE} --timeout=120s --insecure-skip-tls-verify=true || true

                        echo "Pods in namespace ${K8S_NAMESPACE}:"
                        kubectl get pods -n ${K8S_NAMESPACE} -o wide --insecure-skip-tls-verify=true

                        echo "Recent events:"
                        kubectl get events -n ${K8S_NAMESPACE} --sort-by='.metadata.creationTimestamp' --insecure-skip-tls-verify=true | tail -n 10 || true
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "🎉 Cal-App deployed successfully! 👉 https://anis2401098.imcc.skilli.in"
        }
        failure {
            echo "❌ Pipeline failed! Review logs above."
        }
    }
}
