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
          		 # Go to repo root
           		 git checkout testrkg

           		 # Discard any local changes to prevent conflicts
         	         git reset --hard origin/testrkg

           		 # Update the image tag in your Kubernetes manifest
         		 sed -i "s|image: .*|image: $DOCKER_USERNAME/product-catalog:${IMAGE_TAG}|" kubernetes/productcatalog/deploy.yaml

           		 # Configure Git user
         		 git config --global user.email "ravikant@gmail.com"
       		         git config --global user.name "Ravikant Gupta"

            	       	# Stage the changes
         		 git add kubernetes/productcatalog/deploy.yaml

        		 # Commit only if there are changes
          		  if ! git diff --cached --quiet; then
             		  git commit -m "[CI]: Update product catalog image tag"
        		  fi

            	       	  # Fetch latest changes from remote
       		          git fetch origin testrkg

            	       	  # Rebase local changes on top of latest remote
          		  git rebase origin/testrkg || git rebase --abort

          		  # Push changes safely
      		          git push https://$TOKEN@github.com/ravikant1983/ultimate-devops-project-demo.git HEAD:testrkg
           		  '''
    		    }
  		  }
		}


    }
}
