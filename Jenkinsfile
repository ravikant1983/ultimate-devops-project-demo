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
        	dir('src/product-catalog') {
            	  sh '''
            	  curl -sSfL https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh | sh -s latest
            	  ../../bin/golangci-lint run
            	  '''
                }
    }
}



	stage('Docker Build & Push') {
    	   steps {
               withCredentials([usernamePassword(
            	   credentialsId: 'dockerhub',
            	   usernameVariable: 'DOCKER_USERNAME',
            	   passwordVariable: 'DOCKER_PASSWORD'
              )]) {
                  sh '''
                  docker login -u $DOCKER_USERNAME -p $DOCKER_PASSWORD
                  DOCKER_BUILDKIT=1 docker build -t $DOCKER_USERNAME/product-catalog:${IMAGE_TAG} src/product-catalog
                  docker push $DOCKER_USERNAME/product-catalog:${IMAGE_TAG}
                  '''
        }
    }
}


stage('Update Kubernetes Manifest') {
    steps {
        script {
            // 1️⃣ Update the image tag in deploy.yaml using only the Docker username (no token)
            sh """
                sed -i 's|image: .*|image: ${DOCKER_USERNAME}/product-catalog:${IMAGE_TAG}|' kubernetes/productcatalog/deploy.yaml
            """

            // 2️⃣ Configure Git (name/email)
            sh """
                git config --global user.email "ravikant@gmail.com"
                git config --global user.name "Ravikant Gupta"
            """

            // 3️⃣ Checkout the branch safely
            sh """
                git fetch origin
                git checkout rkgtest
            """

            // 4️⃣ Commit only the manifest change
            sh """
                git add kubernetes/productcatalog/deploy.yaml
                git commit -m "[CI]: Update product catalog image tag ${IMAGE_TAG}" || echo "No changes to commit"
            """

            // 5️⃣ Push using Jenkins Git credentials (secure, token stored in Jenkins)
            withCredentials([usernamePassword(credentialsId: 'github-token', 
                                             usernameVariable: 'GIT_USER', 
                                             passwordVariable: 'GIT_TOKEN')]) {
                sh """
                    git remote set-url origin https://${GIT_USER}:${GIT_TOKEN}@github.com/ravikant1983/ultimate-devops-project-demo.git
                    git pull --rebase origin rkgtest || echo "No remote changes to rebase"
                    git push origin rkgtest
                """
            }
        }
    }
}




    }
}
