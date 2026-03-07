pipeline {
    agent any

    environment {
        DOCKER_USERNAME = credentials('dockerhub')
        IMAGE_TAG = "${BUILD_NUMBER}"
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
                script {
                    def imageTag = "${env.BUILD_NUMBER}"
                    def imageName = "${DOCKER_USERNAME}/product-catalog:${imageTag}"
                 
                    echo "Building Docker image: ${imageName}"                   

            	    // Build the Docker image
                    sh "docker build -t ${imageName} src/product-catalog"

                    // Login to Docker Hub
                    sh "echo ${DOCKER_PASSWORD} | docker login -u ${DOCKER_USERNAME} --password-stdin"

                    // Push the image
                    sh "docker push ${imageName}"
            }
        }
    }
}
        stage('Update Kubernetes Manifest') {
            steps {
                withCredentials([string(credentialsId: 'github-token', variable: 'TOKEN')]) {
                    sh '''
                    sed -i "s|image: .*|image: $DOCKER_USERNAME/product-catalog:${IMAGE_TAG}|" kubernetes/productcatalog/deploy.yaml

                    git config --global user.email "ravikant@gmail.com"
                    git config --global user.name "Ravikant Gupta"

                    git add kubernetes/productcatalog/deploy.yaml
                    git commit -m "[CI]: Update product catalog image tag"
                    git push https://$TOKEN@github.com/ravikant1983/ultimate-devops-project-demo.git HEAD:testrkg
                    '''
                }
            }
        }

    }
}
