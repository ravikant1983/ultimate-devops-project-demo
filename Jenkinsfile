pipeline {
    agent any

    environment {
        IMAGE_TAG = "${BUILD_NUMBER}" // Image tag based on Jenkins build number
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/ravikant1983/ultimate-devops-project-demo.git'
            }
        }

        stage('Build') {
            steps {
                sh '''
                cd src/product-catalog
                go mod download
                go build -o product-catalog-service main.go
                '''
            }
        }

        stage('Unit Tests') {
            steps {
                sh '''
                cd src/product-catalog
                go test ./...
                '''
            }
        }

        stage('Code Quality') {
            steps {
                sh '''
                cd src/product-catalog
                go mod tidy
                curl -sSfL https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh | sh -s latest
                ./bin/golangci-lint run .
                '''
            }
        }

        stage('Docker Build & Push') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {
                    sh '''
                    # Login to Docker Hub safely
                    echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin

                    # Build the Docker image
                    docker build -t $DOCKER_USERNAME/product-catalog:$BUILD_NUMBER src/product-catalog

                    # Push the Docker image
                    docker push $DOCKER_USERNAME/product-catalog:$BUILD_NUMBER
                    '''
                }
            }
        }

        stage('Update Kubernetes Manifest') {
            steps {
                withCredentials([string(credentialsId: 'github-token', variable: 'TOKEN')]) {
                    sh '''
                    # Ensure branch is up-to-date
                    git fetch origin testrkg
                    git checkout testrkg
                    git reset --hard origin/testrkg

                    # Update Kubernetes manifest safely
                    sed -i "s|image: .*|image: $DOCKER_USERNAME/product-catalog:$BUILD_NUMBER|" kubernetes/productcatalog/deploy.yaml

                    # Configure Git for CI
                    git config user.email "ravikant@gmail.com"
                    git config user.name "Ravikant Gupta"

                    # Commit and push only if changes exist
                    if ! git diff --quiet; then
                        git add kubernetes/productcatalog/deploy.yaml
                        git commit -m "[CI]: Update product catalog image tag $BUILD_NUMBER"
                        git push https://$TOKEN@github.com/ravikant1983/ultimate-devops-project-demo.git HEAD:testrkg
                    else
                        echo "No changes to commit"
                    fi
                    '''
                }
            }
        }
    }
}
