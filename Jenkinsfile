pipeline {
    agent any
    
    environment {
        FRONTEND_REPO = 'https://github.com/anshjindal/wouessi-ems-frontend.git'
        BACKEND_REPO = 'https://github.com/anshjindal/api-core-backend.git'
        DEPLOYMENT_REPO = 'https://github.com/IshKevin/ems-deploy.git'
        HARBOR_URL = 'localhost:8081'
        PROJECT = 'ems'
        
        GIT_USER_NAME = 'ishkevin'
        GIT_USER_EMAIL = 'carterk279@gmail.com'
        
        
        FRONTEND_CHANGED = 'false'
        BACKEND_CHANGED = 'false'
    }
    
    stages {
        stage('Checkout Repositories') {
            steps {
                script {
                   
                    sh 'mkdir -p frontend backend deployment'
                    
                    
                    try {
                        dir('frontend') {
                            git url: "${FRONTEND_REPO}", branch: 'main'
                        }
                        dir('backend') {
                            git url: "${BACKEND_REPO}", branch: 'main'
                        }
                        dir('deployment') {
                            git url: "${DEPLOYMENT_REPO}", branch: 'main'
                        }
                    } catch (Exception e) {
                        error "Failed to checkout repositories: ${e.message}"
                    }
                }
            }
        }
        
        stage('Detect Changes') {
            steps {
                script {
                   
                    echo "Setting up for conditional builds..."
                    
                    
                    if (currentBuild.number == 1) {
                        echo "First build detected - will build both frontend and backend"
                        env.FRONTEND_CHANGED = 'true'
                        env.BACKEND_CHANGED = 'true'
                    } else {
                        try {
                            
                            dir('frontend') {
                                def frontendChanges = sh(
                                    script: "git log -n 1 --oneline",
                                    returnStdout: true
                                ).trim()
                                
                                echo "Latest frontend commit: ${frontendChanges}"
                                if (frontendChanges) {
                                    env.FRONTEND_CHANGED = 'true'
                                }
                            }
                            
                            
                            dir('backend') {
                                def backendChanges = sh(
                                    script: "git log -n 1 --oneline",
                                    returnStdout: true
                                ).trim()
                                
                                echo "Latest backend commit: ${backendChanges}"

                                if (backendChanges) {
                                    env.BACKEND_CHANGED = 'true'
                                }
                            }
                        } catch (Exception e) {
                            echo "Error during change detection: ${e.message}"
                            echo "Defaulting to building both components for safety"
                            env.FRONTEND_CHANGED = 'true'
                            env.BACKEND_CHANGED = 'true'
                        }
                    }
                    
                    echo "Build status: Frontend=${env.FRONTEND_CHANGED}, Backend=${env.BACKEND_CHANGED}"
                }
            }
        }
        
        stage('Build Frontend Image') {
            when {
                expression { return env.FRONTEND_CHANGED == 'true' }
            }
            steps {
                echo "Building frontend image..."
                
                script {
                    def useKaniko = false
                    try {
                        sh "which /kaniko/executor || echo 'Kaniko not found'"
                        useKaniko = fileExists('/kaniko/executor')
                    } catch (Exception e) {
                        echo "Kaniko not detected: ${e.message}"
                    }
                    
                    if (useKaniko) {
                        withCredentials([usernamePassword(credentialsId: 'harbor-credentials', usernameVariable: 'HARBOR_USERNAME', passwordVariable: 'HARBOR_PASSWORD')]) {
                            sh '''
                            # Create Docker config for authentication
                            mkdir -p /kaniko/.docker
                            cat > /kaniko/.docker/config.json << EOF
{
    "auths": {
        "${HARBOR_URL}": {
            "username": "${HARBOR_USERNAME}",
            "password": "${HARBOR_PASSWORD}"
        }
    }
}
EOF
                            
                            # Run Kaniko to build and push frontend image
                            /kaniko/executor \
                                --context=./frontend \
                                --dockerfile=./frontend/Dockerfile \
                                --destination=${HARBOR_URL}/${PROJECT}/frontend:${BUILD_NUMBER} \
                                --destination=${HARBOR_URL}/${PROJECT}/frontend:latest \
                                --skip-tls-verify
                            '''
                        }
                    } else {
                    
                        withCredentials([usernamePassword(credentialsId: 'harbor-credentials', usernameVariable: 'HARBOR_USERNAME', passwordVariable: 'HARBOR_PASSWORD')]) {
                            sh '''
                            # Login to Harbor
                            echo "${HARBOR_PASSWORD}" | docker login ${HARBOR_URL} -u ${HARBOR_USERNAME} --password-stdin
                            
                            # Build and push image
                            cd frontend
                            docker build -t ${HARBOR_URL}/${PROJECT}/frontend:${BUILD_NUMBER} -t ${HARBOR_URL}/${PROJECT}/frontend:latest .
                            docker push ${HARBOR_URL}/${PROJECT}/frontend:${BUILD_NUMBER}
                            docker push ${HARBOR_URL}/${PROJECT}/frontend:latest
                            '''
                        }
                    }
                }
                echo "Frontend build completed successfully"
            }
        }
        
        stage('Build Backend Image') {
            when {
                expression { return env.BACKEND_CHANGED == 'true' }
            }
            steps {
                echo "Building backend image..."
                
                script {
                    def useKaniko = false
                    try {
                        sh "which /kaniko/executor || echo 'Kaniko not found'"
                        useKaniko = fileExists('/kaniko/executor')
                    } catch (Exception e) {
                        echo "Kaniko not detected: ${e.message}"
                    }
                    
                    if (useKaniko) {
                        withCredentials([usernamePassword(credentialsId: 'harbor-credentials', usernameVariable: 'HARBOR_USERNAME', passwordVariable: 'HARBOR_PASSWORD')]) {
                            sh '''
                            # Create Docker config for authentication
                            mkdir -p /kaniko/.docker
                            cat > /kaniko/.docker/config.json << EOF
{
    "auths": {
        "${HARBOR_URL}": {
            "username": "${HARBOR_USERNAME}",
            "password": "${HARBOR_PASSWORD}"
        }
    }
}
EOF
                            
                            # Run Kaniko to build and push backend image
                            /kaniko/executor \
                                --context=./backend \
                                --dockerfile=./backend/Dockerfile \
                                --destination=${HARBOR_URL}/${PROJECT}/backend:${BUILD_NUMBER} \
                                --destination=${HARBOR_URL}/${PROJECT}/backend:latest \
                                --skip-tls-verify
                            '''
                        }
                    } else {
                        withCredentials([usernamePassword(credentialsId: 'harbor-credentials', usernameVariable: 'HARBOR_USERNAME', passwordVariable: 'HARBOR_PASSWORD')]) {
                            sh '''
                            # Login to Harbor
                            echo "${HARBOR_PASSWORD}" | docker login ${HARBOR_URL} -u ${HARBOR_USERNAME} --password-stdin
                            
                            # Build and push image
                            cd backend
                            docker build -t ${HARBOR_URL}/${PROJECT}/backend:${BUILD_NUMBER} -t ${HARBOR_URL}/${PROJECT}/backend:latest .
                            docker push ${HARBOR_URL}/${PROJECT}/backend:${BUILD_NUMBER}
                            docker push ${HARBOR_URL}/${PROJECT}/backend:latest
                            '''
                        }
                    }
                }
                echo "Backend build completed successfully"
            }
        }
        
        stage('Update Helm Charts') {
            when {
                expression { 
                    return env.FRONTEND_CHANGED == 'true' || env.BACKEND_CHANGED == 'true' 
                }
            }
            steps {
                dir('deployment') {
                    withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
                        sh """
                        git config user.email "${GIT_USER_EMAIL}"
                        git config user.name "${GIT_USER_NAME}"
                        
                        # Update only the charts for components that were built
                        if [ "${env.FRONTEND_CHANGED}" = "true" ]; then
                            echo "Updating frontend Helm chart..."
                            sed -i "s|tag: \\".*\\"|tag: \\"${BUILD_NUMBER}\\"|g" charts/ems-frontend/values.yaml
                            git add charts/ems-frontend/values.yaml
                        fi
                        
                        if [ "${env.BACKEND_CHANGED}" = "true" ]; then
                            echo "Updating backend Helm chart..."
                            sed -i "s|tag: \\".*\\"|tag: \\"${BUILD_NUMBER}\\"|g" charts/ems-backend/values.yaml
                            git add charts/ems-backend/values.yaml
                        fi
                        
                        # Commit changes if any
                        if git diff --cached --quiet; then
                            echo "No changes to commit"
                        else
                            git commit -m "CI: Update image tags to build ${BUILD_NUMBER}"
                            
                            # Push changes
                            git remote set-url origin https://${GIT_USER_NAME}:${GITHUB_TOKEN}@github.com/IshKevin/ems-deploy.git
                            git push origin main
                            echo "Successfully updated Helm charts and pushed changes"
                        fi
                        """
                    }
                }
            }
        }
    }
    
    post {
        always {
            // Clean workspace to avoid clutter
            cleanWs()
        }
        failure {
            echo "Pipeline failed - check stage logs for details"
            // Add notification logic like Slack, email, etc.
        }
        success {
            echo "Pipeline completed successfully"
            // Add notification logic like Slack, email, etc.
        }
    }
}