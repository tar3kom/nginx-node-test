pipeline {
    agent any

    tools {
        nodejs 'node20'
    }

    triggers {
        githubPush()
    }

    environment {
        IMAGE_NAME = "tar3kom/nginx-node-test"
        DEPLOY_BRANCH = "deploy"
        DEPLOY_PATH   = "deploy-repo"
        DEPLOY_REPO = "github.com/tar3kom/nginx-node-test.git"
        ARGO_APP      = "mnsp-frontend-dashboard"
        ARGO_URL      = "https://mnsp-argocd.inet.co.th"
    }

    stages {

        // ----------------------
        // Checkout (Tag only)
        // ----------------------
        stage('Checkout') {
            when { buildingTag() }
            steps {
                checkout scm
                script {
                    env.TAG = sh(
                        script: 'git describe --tags',
                        returnStdout: true
                    ).trim()
                }
                echo "🏷 Deploy tag = ${TAG}"
            }
        }

        // ----------------------
        // Install & Test
        // ----------------------
        stage('Install Dependencies') {
            when { buildingTag() }
            steps {
                sh 'npm ci'
            }
        }

        stage('Test') {
            when { buildingTag() }
            steps {
                sh 'npm test'
            }
        }

        // ----------------------
        // Build & Push Docker Image
        // ----------------------
        stage('Build Docker Image') {
            when { buildingTag() }
            steps {
                sh """
                    docker build \
                      -t ${IMAGE_NAME}:${TAG} \
                      -t ${IMAGE_NAME}:latest .
                """
            }
        }

        stage('Push Docker Image') {
            when { buildingTag() }
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'docker-cred',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {
                    sh """
                        echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin
                        docker push ${IMAGE_NAME}:${TAG}
                        docker push ${IMAGE_NAME}:latest
                        docker logout
                    """
                }
            }
        }

        // ----------------------
        // GitOps Deploy (branch deploy)
        // ----------------------
        stage('GitOps Deploy') {
            when { buildingTag() }
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'gitops-cred',
                        usernameVariable: 'GIT_USER',
                        passwordVariable: 'GIT_TOKEN'
                    )
                ]) {
                    sh '''
                        set -e
                        echo "🚀 GitOps deploy: update image tag"

                        git clone --branch ${DEPLOY_BRANCH} \
                          https://${GIT_USER}:${GIT_TOKEN}@${DEPLOY_REPO} \
                          ${DEPLOY_PATH}

                        cd ${DEPLOY_PATH}

                        sed -i "/^image:/,/^[^ ]/ s|^\\s*tag: .*|  tag: ${TAG}|" values.yaml

                        git config user.name "jenkins[bot]"
                        git config user.email "jenkins[bot]@example.com"

                        git add values.yaml
                        git commit -m "chore: bump image tag ${TAG}" || echo "No changes"
                        git push origin ${DEPLOY_BRANCH}

                        echo "✅ GitOps updated to tag ${TAG}"
                    '''
                }
            }
        }

        // ----------------------
        // ArgoCD Sync
        // ----------------------
        stage('ArgoCD Sync') {
            when { buildingTag() }
            steps {
                withCredentials([
                    string(
                        credentialsId: 'argocd-token',
                        variable: 'ARGOCD_TOKEN'
                    )
                ]) {
                    sh '''
                        set -e
                        echo "🚀 Triggering ArgoCD sync"

                        curl -k -X POST \
                          -H "Authorization: Bearer ${ARGOCD_TOKEN}" \
                          -H "Content-Type: application/json" \
                          ${ARGO_URL}/api/v1/applications/${ARGO_APP}/sync \
                          -d '{}'

                        echo "✅ ArgoCD sync triggered"
                    '''
                }
            }
        }
    }

    post {
        success {
            echo '✅ Tag deploy success'
        }
        failure {
            echo '❌ Tag deploy failed'
        }
        always {
            echo '📦 Pipeline finished'
        }
    }
}
