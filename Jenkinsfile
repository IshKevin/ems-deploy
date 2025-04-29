pipeline {
    agent {
        docker {
            image 'localhost:8081/ems/kaniko-compose:debug'
            args '--entrypoint="" -v /var/run/docker.sock:/var/run/docker.sock -u root'
        }
    }
    environment {
        FRONTEND_REPO = 'https://github.com/anshjindal/wouessi-ems-frontend.git'
        BACKEND_REPO = 'https://github.com/anshjindal/api-core-backend.git'
        DEPLOYMENT_REPO = 'https://github.com/anshjindal/ems-deployment.git'
        HARBOR_URL = 'localhost:8081'
        PROJECT = 'ems'
        DOCKER_CONFIG = '/kaniko/.docker/config.json'
        GIT_USER_NAME = 'anshjindal'
        GIT_USER_EMAIL = 'ansjindal@wouessi.com'
    }
    stages {
        stage('Checkout Repositories') {
            steps {
                dir('frontend') {
                    git url: "${FRONTEND_REPO}", branch: 'main'
                }
                dir('backend') {
                    git url: "${BACKEND_REPO}", branch: 'main'
                }
                dir('deployment') {
                    git url: "${DEPLOYMENT_REPO}", branch: 'main'
                }
            }
        }
        stage('Setup Harbor Credentials') {
            steps {
                withCredentials([file(credentialsId: 'harbor-credentials', variable: 'DOCKER_CONFIG_FILE')]) {
                    sh """
                    mkdir -p /kaniko/.docker
                    cp \$DOCKER_CONFIG_FILE /kaniko/.docker/config.json
                    chmod 600 /kaniko/.docker/config.json
                    """
                }
            }
        }
        stage('Build and Push Frontend') {
            steps {
                dir('frontend') {
                    sh """
                    /kaniko/executor \
                    --context `pwd` \
                    --dockerfile `pwd`/Dockerfile \
                    --destination ${HARBOR_URL}/${PROJECT}/frontend:${BUILD_NUMBER} \
                    --destination ${HARBOR_URL}/${PROJECT}/frontend:latest \
                    --cache=true \
                    --insecure
                    """
                }
            }
        }
        stage('Build and Push Backend') {
            steps {
                dir('backend') {
                    sh """
                    /kaniko/executor \
                    --context `pwd` \
                    --dockerfile `pwd`/Dockerfile \
                    --destination ${HARBOR_URL}/${PROJECT}/backend:${BUILD_NUMBER} \
                    --destination ${HARBOR_URL}/${PROJECT}/backend:latest \
                    --cache=true \
                    --insecure
                    """
                }
            }
        }
        stage('Update Helm Charts') {
            steps {
                dir('deployment') {
                    withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
                        sh """
                        git config user.email "${GIT_USER_EMAIL}"
                        git config user.name "${GIT_USER_NAME}"
                        sed -i "s|tag: \".*\"|tag: \"${BUILD_NUMBER}\"|g" charts/ems-frontend/values.yaml
                        sed -i "s|tag: \".*\"|tag: \"${BUILD_NUMBER}\"|g" charts/ems-backend/values.yaml
                        git add charts/ems-frontend/values.yaml charts/ems-backend/values.yaml
                        git commit -m "Update Helm chart image tags to ${BUILD_NUMBER}"
                        git push https://${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/ems-deployment.git HEAD:main
                        """
                    }
                }
            }
        }
    }
    post {
        always {
            node('') {
                sh """
                docker system prune -f || true
                """
            }
        }
        failure {
            echo "Pipeline failed. Check logs for details."
        }
    }
}