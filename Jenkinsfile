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
    }
    
    stages {
        stage('Checkout Repositories') {
            steps {
                sh 'mkdir -p frontend backend deployment'
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
        
        stage('Build Frontend Image') {
            steps {
                dir('frontend') {
                    // Create a temp directory
                    sh 'mkdir -p /tmp/frontend-build'
                    
                    // Create a shell script to build the image using buildah
                    sh '''
                    # Configure buildah to use vfs storage driver (user-specific)
                    mkdir -p ~/.config/containers
                    cat << EOF > ~/.config/containers/storage.conf
[storage]
driver = "vfs"
runroot = "/var/run/containers/storage"
graphroot = "/var/lib/containers/storage"
EOF

                    # Clean up existing buildah storage
                    buildah rm --all || true
                    buildah rmi --all || true
                    rm -rf ~/.local/share/containers/* || true

                    # Ensure permissions for storage directory
                    mkdir -p /var/lib/containers/storage
                    chmod -R 755 /var/lib/containers/storage

                    cat > /tmp/frontend-build/build.sh << 'EOF'
#!/bin/bash
set -ex
echo "Debug: Kernel version: $(uname -r)"
echo "Debug: Buildah version: $(buildah --version || echo 'buildah not installed')"
echo "Debug: Storage driver: $(buildah info --format '{{.store.GraphDriverName}}' || echo 'failed to get storage driver')"
echo "Debug: Storage config (user): $(cat ~/.config/containers/storage.conf || echo 'no user config')"

# Install buildah if not present
if ! command -v buildah &> /dev/null; then
    echo "Installing buildah..."
    apt-get update && apt-get install -y buildah || \
    (apt-get update && apt-get install -y software-properties-common && \
    add-apt-repository -y ppa:projectatomic/ppa && \
    apt-get update && apt-get install -y buildah)
fi

# Verify storage driver
if [ "$(buildah info --format '{{.store.GraphDriverName}}')" != "vfs" ]; then
    echo "Error: Storage driver is not vfs"
    exit 1
fi

# Set variables
SRC_DIR=$1
REGISTRY=$2
PROJECT=$3
TAG=$4
IMAGE_NAME="${REGISTRY}/${PROJECT}/frontend:${TAG}"
LATEST_IMAGE="${REGISTRY}/${PROJECT}/frontend:latest"

echo "Building image: ${IMAGE_NAME}"
cd ${SRC_DIR}

# Build the image
CONTAINER=$(buildah from node:18-alpine)
buildah copy $CONTAINER . /app
buildah config --workingdir /app $CONTAINER
buildah run $CONTAINER npm install
buildah run $CONTAINER npm run build
buildah config --entrypoint '["npm", "start"]' $CONTAINER
buildah commit $CONTAINER ${IMAGE_NAME}
buildah tag ${IMAGE_NAME} ${LATEST_IMAGE}

# Push the image
echo "Pushing image: ${IMAGE_NAME}"
buildah push --tls-verify=false ${IMAGE_NAME} docker://${IMAGE_NAME}
buildah push --tls-verify=false ${LATEST_IMAGE} docker://${LATEST_IMAGE}

echo "Frontend image build and push completed"
EOF
                    chmod +x /tmp/frontend-build/build.sh
                    '''
                    
                    // Set up Docker registry auth config for buildah
                    withCredentials([usernamePassword(credentialsId: 'harbor-credentials', usernameVariable: 'HARBOR_USERNAME', passwordVariable: 'HARBOR_PASSWORD')]) {
                        sh '''
                        mkdir -p ~/.docker
                        cat > ~/.docker/config.json << EOF
{
    "auths": {
        "${HARBOR_URL}": {
            "username": "${HARBOR_USERNAME}",
            "password": "${HARBOR_PASSWORD}"
        }
    }
}
EOF
                        '''
                    }
                    
                    // Run the build script
                    sh '''
                    /tmp/frontend-build/build.sh $(pwd) ${HARBOR_URL} ${PROJECT} ${BUILD_NUMBER}
                    '''
                }
            }
        }
        
        stage('Build Backend Image') {
            steps {
                dir('backend') {
                    // Create a temp directory
                    sh 'mkdir -p /tmp/backend-build'
                    
                    // Create a shell script to build the image using buildah
                    sh '''
                    # Configure buildah to use vfs storage driver (user-specific)
                    mkdir -p ~/.config/containers
                    cat << EOF > ~/.config/containers/storage.conf
[storage]
driver = "vfs"
runroot = "/var/run/containers/storage"
graphroot = "/var/lib/containers/storage"
EOF

                    # Clean up existing buildah storage
                    buildah rm --all || true
                    buildah rmi --all || true
                    rm -rf ~/.local/share/containers/* || true

                    # Ensure permissions for storage directory
                    mkdir -p /var/lib/containers/storage
                    chmod -R 755 /var/lib/containers/storage

                    cat > /tmp/backend-build/build.sh << 'EOF'
#!/bin/bash
set -ex
echo "Debug: Kernel version: $(uname -r)"
echo "Debug: Buildah version: $(buildah --version || echo 'buildah not installed')"
echo "Debug: Storage driver: $(buildah info --format '{{.store.GraphDriverName}}' || echo 'failed to get storage driver')"
echo "Debug: Storage config (user): $(cat ~/.config/containers/storage.conf || echo 'no user config')"

# Install buildah if not present
if ! command -v buildah &> /dev/null; then
    echo "Installing buildah..."
    apt-get update && apt-get install -y buildah || \
    (apt-get update && apt-get install -y software-properties-common && \
    add-apt-repository -y ppa:projectatomic/ppa && \
    apt-get update && apt-get install -y buildah)
fi

# Verify storage driver
if [ "$(buildah info --format '{{.store.GraphDriverName}}')" != "vfs" ]; then
    echo "Error: Storage driver is not vfs"
    exit 1
fi

# Set variables
SRC_DIR=$1
REGISTRY=$2
PROJECT=$3
TAG=$4
IMAGE_NAME="${REGISTRY}/${PROJECT}/backend:${TAG}"
LATEST_IMAGE="${REGISTRY}/${PROJECT}/backend:latest"

echo "Building image: ${IMAGE_NAME}"
cd ${SRC_DIR}

# Build the image
CONTAINER=$(buildah from node:18-alpine)
buildah copy $CONTAINER . /app
buildah config --workingdir /app $CONTAINER
buildah run $CONTAINER npm install
buildah config --entrypoint '["npm", "start"]' $CONTAINER
buildah commit $CONTAINER ${IMAGE_NAME}
buildah tag ${IMAGE_NAME} ${LATEST_IMAGE}

# Push the image
echo "Pushing image: ${IMAGE_NAME}"
buildah push --tls-verify=false ${IMAGE_NAME} docker://${IMAGE_NAME}
buildah push --tls-verify=false ${LATEST_IMAGE} docker://${LATEST_IMAGE}

echo "Backend image build and push completed"
EOF
                    chmod +x /tmp/backend-build/build.sh
                    '''
                    
                    // Run the build script
                    sh '''
                    /tmp/backend-build/build.sh $(pwd) ${HARBOR_URL} ${PROJECT} ${BUILD_NUMBER}
                    '''
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
                        
                        # Update the image tags in Helm charts
                        sed -i "s|tag: \\\".*\\\"|tag: \\\"${BUILD_NUMBER}\\\"|g" charts/ems-frontend/values.yaml
                        sed -i "s|tag: \\\".*\\\"|tag: \\\"${BUILD_NUMBER}\\\"|g" charts/ems-backend/values.yaml
                        
                        # Commit and push changes
                        git add charts/ems-frontend/values.yaml charts/ems-backend/values.yaml
                        git commit -m "Update Helm chart image tags to ${BUILD_NUMBER}"
                        
                        # Use HTTPS with token for authentication instead of SSH
                        git remote set-url origin https://${GIT_USER_NAME}:${GITHUB_TOKEN}@github.com/IshKevin/ems-deploy.git
                        git push origin main
                        """
                    }
                }
            }
        }
    }
    
    post {
        always {
            sh '''
            # Clean up temporary directories
            rm -rf /tmp/frontend-build /tmp/backend-build
            '''
        }
        failure {
            echo "Pipeline failed. Check logs for details."
        }
    }
}