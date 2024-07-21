pipeline {
    agent any
    tools {
        nodejs 'Node 20'
    }
    environment {
        GH_TOKEN = credentials('github-pat')
        REPO_NAME = "helm-eks-autoscaler"
        REPO_OWNER = "cyse7125-su24-team12" // Define the repository owner
        DOCKERHUB_REPO_AUTOSCALER = 'bala699/cluster-autoscaler'
        DOCKERHUB_REPO_METRICS = 'bala699/metrics-server'
        BUILD_NUMBER = 'latest'
        DOCKER_BUILDER_NAME = 'db-autoscaler-builder'
        // CHANGE THESE CREDS
        DOCKER_AUTOSCALER_FILE = 'Dockerfile.autoscaler'
        DOCKER_METRICS_FILE = 'Dockerfile.metrics'
    }
    stages {
        stage('Install helm') {
            steps {
                script {
                    sh '''
                        if ! command -v helm &> /dev/null; then
                            echo "Helm could not be found, installing Helm."

                            # Add the GPG key for the official Helm stable repository
                            curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null

                            # Install apt-transport-https to allow the use of a repository accessed via HTTPS
                            sudo apt-get install apt-transport-https --yes

                            # Add the Helm repository
                            echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list

                            # Update apt package index
                            sudo apt-get update

                            # Install Helm
                            sudo apt-get install helm

                            echo "Helm installation is complete."
                        else
                            echo "Helm is already installed."
                        fi
                    '''
                }
            }
        }
        stage ('check helm lint and template')
        {
            when {
                expression {
                    // multibranch pipeline
                    return env.BRANCH_NAME != null
                }
            }
            steps {
                script {
                    sh '''
                        
                        helm lint . 
                        if [ $? -ne 0 ]; then
                            echo "Helm lint failed"
                            exit 1
                        fi
                        helm template .
                        if [ $? -ne 0 ]; then
                            echo "Helm template failed"
                            exit 1
                        fi
                    '''
                }
            }
        }
        stage('Setup Commitlint') {
            when {
                expression {
                    // Check if the BRANCH_NAME is null
                    return env.BRANCH_NAME != null
                }
            }
            steps {
                sh """
        # Check if commitlint is already installed and install if not
        if ! npm list -g @commitlint/cli | grep -q '@commitlint/cli'; then
            npm install -g @commitlint/cli
        fi

        if ! npm list -g @commitlint/config-conventional | grep -q '@commitlint/config-conventional'; then
            npm install -g @commitlint/config-conventional
        fi

        # Ensure the commitlint config file is present
        if [ ! -f commitlint.config.js ]; then
            echo "module.exports = {extends: ['@commitlint/config-conventional']}" > commitlint.config.js
        fi
        """
            }
        }
        stage('Lint commit messages') {
            when {
                expression {
                    // Check if the BRANCH_NAME is null
                    return env.BRANCH_NAME != null
                }
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'git-credentials-id', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
                    sh '''
                node --version
                echo " source branch: $CHANGE_BRANCH"
                echo " target branch: $CHANGE_TARGET"
                echo " url: $CHANGE_URL"

                # Extract the owner and repository name from the CHANGE_URL
                OWNER=$(echo "$CHANGE_URL" | sed 's|https://github.com/\\([^/]*\\)/\\([^/]*\\)/pull/.*|\\1|')
                REPO=$(echo "$CHANGE_URL" | sed 's|https://github.com/\\([^/]*\\)/\\([^/]*\\)/pull/.*|\\2|')

                # Extract the pull request number from the CHANGE_URL
                PR_NUMBER=$(echo "$CHANGE_URL" | sed 's|.*/pull/\\([0-9]*\\).*|\\1|')

                echo "Owner: $OWNER"
                echo "Repository: $REPO"
                echo "Pull Request Number: $PR_NUMBER"

                # GitHub API endpoint to get commits from a specific pull request
                API_URL="https://api.github.com/repos/$OWNER/$REPO/pulls/$PR_NUMBER/commits"

                # Make an authenticated API request to get the commits
                COMMITS=$(curl -s -H "Authorization: token $GIT_PASSWORD" "$API_URL")

                echo "$COMMITS" | jq -c '.[]' | while IFS= read -r COMMIT; do
                    # Extract the commit message from each commit JSON object
                    COMMIT_MESSAGE=$(echo "$COMMIT" | jq -r '.commit.message')


                    # Echo and lint the commit message
                    echo "Linting message: $COMMIT_MESSAGE"
                    echo "$COMMIT_MESSAGE" | npx commitlint
                    if [ $? -ne 0 ]; then
                        echo "Commit message linting failed."
                        exit 1
                    fi
                done
                '''
                }
            }
        }
        stage('Checkout') {
            when {
                expression {
                    // Check if the BRANCH_NAME is null
                    return env.BRANCH_NAME == null
                }
            }
            steps {
            checkout([$class: 'GitSCM',
            branches: [[name: '*/main']],
            extensions: [[$class: 'CleanCheckout']],
            userRemoteConfigs: [[url: 'https://github.com/cyse7125-su24-team12/helm-eks-autoscaler.git', credentialsId: 'git-credentials-id']]
                ])
            }
        }
        stage('Check for [skip ci] tag in commit message'){
            when{
                expression{
                    return env.BRANCH_NAME == null
                }
            }
            steps{
                script{
                    // Checking the last commit message for the '[skip ci]' tag
                    result = sh(script: "git log -1 --pretty=%B | grep '\\[skip ci\\]'", returnStatus: true)
                    if (result == 0) {
                        echo "Commit message contains '[skip ci]', skipping CI process."
                        currentBuild.result = 'ABORTED'
                        error('CI process skipped due to [skip ci] tag in commit message.')
                    } else {
                        echo "No [skip ci] tag found, proceeding with build."
                        // Additional steps to continue the build can be placed here
                    }
                }
            }
        }
        stage('Setup semantic,github-release & yq'){
            when{
                expression{
                    return env.BRANCH_NAME == null
                }
            }
            steps{
                script {
                    sh '''
                        npm install -g \
                        semantic-release \
                        @semantic-release/changelog \
                        @semantic-release/github \
                        @semantic-release/commit-analyzer \
                        @semantic-release/release-notes-generator \
                        @semantic-release/exec \
                        @semantic-release/git 
                        sudo apt update 
                        sudo apt install yq -y 
                        ls -a 
                        npm install -g github-release-cli
                    '''
                }
            }
        }
        stage(' semantic release'){
            when{
                expression{
                    return env.BRANCH_NAME == null
                }
            }
            steps{
                script{
                    writeFile file: '.releaserc', text: '''
                    {
                        "branches": ["main"],
                        "plugins": [
                            "@semantic-release/commit-analyzer",
                            "@semantic-release/release-notes-generator",
                            "@semantic-release/changelog",
                            [
                                "@semantic-release/exec", 
                                {
                                    "publishCmd": "helm package .  --version ${nextRelease.version}",
                                    "prepareCmd": "sed -i 's/version:.*/version: ${nextRelease.version}/' Chart.yaml"
                                }
                            ],
                            [
                                "@semantic-release/git", 
                                {
                                    "assets": ["Chart.yaml"],
                                    "message": "chore(release): ${nextRelease.version} [skip ci]"
                                }
                            ],
                            [
                                "@semantic-release/github",
                                {
                                    "assets": [
                                        { "path": "./*.tgz"},
                                    ]
                                }
                            ]
                        ]
                    }
                    '''
                    sh '''
                    cat ./.releaserc
                    npx semantic-release
                    ls -a 
                    rm -rf ./*.tgz
                    '''
                }
            }
        }
        stage('Github release edit')
        {
            when{
                expression{
                    return env.BRANCH_NAME == null
                }
            }
            steps{
                withCredentials([string(credentialsId: 'github-pat', variable: 'GH_TOKEN')]) {
                    script{
                        sh '''
                            echo "Creating a new release"
                            RELEASE_VERSION=$(yq -r '.version' Chart.yaml)
                            CHART_NAME=$(yq -r '.name' Chart.yaml)
                            echo $RELEASE_VERSION
                            echo $NEW_RELEASE_VERSION
                            export GITHUB_TOKEN=$GH_TOKEN
                            release_id=$(github-release list --owner $REPO_OWNER  --repo $REPO_NAME | head -n 1 | egrep -o 'id=[0-9]+' | cut -d '=' -f 2)
                            release_tag=$(github-release list --owner $REPO_OWNER --repo $REPO_NAME | head -n 1 | egrep -o 'tag_name="[^"]+"' | cut -d '"' -f 2)

                            echo "The extracted release ID is: $release_id"
                            echo "The extracted release tag is: $release_tag"
                            new_release_name="$CHART_NAME-$RELEASE_VERSION"
                            echo "The new release name is: $new_release_name"

                            github-release upload --owner $REPO_OWNER --repo $REPO_NAME --release-id $release_id --release-name $new_release_name 
                        '''
                    }
                }
            }
        }
        stage('Setup Buildx') {
            when {
                expression {
                    // Check if the BRANCH_NAME is null
                    return env.BRANCH_NAME == null
                }
            }
            steps {
                script {
                    sh '''
                        mkdir -p ~/.docker/cli-plugins/
                        curl -sL https://github.com/docker/buildx/releases/download/v0.14.1/buildx-v0.14.1.linux-amd64 -o ~/.docker/cli-plugins/docker-buildx
                        chmod +x ~/.docker/cli-plugins/docker-buildx
                        export PATH=$PATH:~/.docker/cli-plugins
                        '''
                }
            }
        }
        stage('Setup hadolint')
        {
            when{
                expression{
                    return env.BRANCH_NAME != null
                }
            }
            steps {
                script {
                        sh '''
                        # Check if Hadolint is already installed and at the desired version
                        if ! command -v hadolint &>/dev/null || [[ "$(hadolint --version)" != *"v2.10.0"* ]]; then
                            echo "Hadolint not found or not the desired version, installing..."
                            
                            # Download Hadolint binary
                            sudo wget -O /usr/local/bin/hadolint https://github.com/hadolint/hadolint/releases/download/v2.10.0/hadolint-Linux-x86_64
                            
                            # Make it executable
                            sudo chmod +x /usr/local/bin/hadolint
                        else
                            echo "Hadolint is already installed and at the correct version."
                        fi

                        # Verify installation
                        hadolint --version
                        '''
                    }
            }
        }
        stage('Lint Dockerfile') {
            when {
                expression {
                    // Check if the BRANCH_NAME is null
                    return env.BRANCH_NAME != null
                }
            }
            steps {
                script {
                    echo 'Linting Dockerfile autoscaler'
                    sh 'hadolint $DOCKER_AUTOSCALER_FILE'
                }
                script{
                    echo 'Linting Dockerfile metics'
                    sh 'hadolint $DOCKER_METRICS_FILE'
                }

            }
        }
        stage('Build and push the docker image using buildx') {
            when {
                expression {
                    // Check if the BRANCH_NAME is null
                    return env.BRANCH_NAME == null
                }
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials-id', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD'),
                    string(credentialsId: 'github-pat', variable: 'GH_TOKEN')]) {
                    script {
                        sh '''
            # Login to Docker Hub
            echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin

            export GITHUB_TOKEN=$GH_TOKEN
            release_id=$(github-release list --owner $REPO_OWNER  --repo $REPO_NAME | head -n 1 | egrep -o 'id=[0-9]+' | cut -d '=' -f 2)
            release_tag=$(github-release list --owner $REPO_OWNER --repo $REPO_NAME | head -n 1 | grep -o 'tag_name="[^"]*"' | cut -d '"' -f 2 | sed 's/^v//')

            echo "The extracted release tag is: $release_tag"
            # Create and use builder instance
            # Check if the builder 'db-autoscaler-builder' already exists
            if ! docker buildx ls | grep -q "${DOCKER_BUILDER_NAME}"; then
                echo "Builder does not exist. Creating builder..."
                # Create the builder
                docker buildx create --name "${DOCKER_BUILDER_NAME}" --driver docker-container
            else
                echo "Builder already exists."
            fi

            # Use the builder
            docker buildx use "${DOCKER_BUILDER_NAME}"

            docker buildx ls

            # Build and push Docker image for the autoscaler
            docker buildx build --platform linux/amd64,linux/arm64 -t ${DOCKERHUB_REPO_AUTOSCALER}:${release_tag} -t ${DOCKERHUB_REPO_AUTOSCALER}:${BUILD_NUMBER} -f ${DOCKER_AUTOSCALER_FILE} --push .

            # Build and push Docker image for the metric
            docker buildx build --platform linux/amd64,linux/arm64 -t ${DOCKERHUB_REPO_METRICS}:${release_tag} -t ${DOCKERHUB_REPO_METRICS}:${BUILD_NUMBER} -f ${DOCKER_METRICS_FILE} --push .

            # Logout from Docker Hub
            docker logout
            '''
                    }
                }
            }
        }
        stage('Docker clean up'){
            when{
                expression{
                    return env.BRANCH_NAME == null
                }
            }
            steps{
                script{
                    sh '''
                        # Check if the builder "${DOCKER_BUILDER_NAME}" exists
                        if docker buildx ls | grep -q "${DOCKER_BUILDER_NAME}"; then
                            echo "Builder exists. Removing builder..."
                            docker buildx rm "${DOCKER_BUILDER_NAME}"
                        else
                            echo "Builder does not exist or already removed."
                        fi
                    '''
                }
            }
        }
    }
    post {
        success {
            echo 'build succeeded!'
        }
        failure {
            echo 'build failed!'
        }
    }
    
}
