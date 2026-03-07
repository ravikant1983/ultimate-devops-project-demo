pipeline {
    agent any

    environment {
        GO_VERSION = "1.22"
        IMAGE_NAME = "ravikant/product-catalog"
        IMAGE_TAG = "${BUILD_NUMBER}"
        DOCKER_CREDENTIALS = credentials('dockerhub')
        GITHUB_TOKEN = credentials('github-token')
        SONAR_TOKEN = credentials('sonar-token')
    }

    options {
        ansiColor('xterm')
        buildDiscarder(logRotator(numToKeepStr: '20'))
        timeout(time: 60, unit: 'MINUTES')
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'cicd', url: 'https://github.com/ravikant1983/ultimate-devops-project-demo.git'
            }
        }


        stage('Build Go App') {
            steps {
                sh """
                cd src/product-catalog
                go mod download
                go build -o product-catalog-service main.go
                """
            }
        }

        stage('Unit Tests') {
            steps {
                sh """
                cd src/product-catalog
                go test ./...
                """
            }
        }

        stage('Code Quality Scan') {
            steps {
                sh """
		cd src/product-catalog
 		go mod tidy
                curl -sSfL https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh | sh -s latest
                ./bin/golangci-lint run .
                """
            }
        }

	stage('SonarQube Scan') {
   	    steps {
        	script {
            	  def scannerHome = tool 'sonar-scanner'
           	  withSonarQubeEnv('sonar-server') {
                	sh """
                	${scannerHome}/bin/sonar-scanner \
               	 	-Dsonar.projectKey=product-catalog \
                	-Dsonar.sources=src/product-catalog
                	"""
            }
        }
    }
}



stage('Docker Build') {
    steps {
        sh """
	cd src/product-catalog
        docker build -t rkg1983/product-catalog:${BUILD_NUMBER} .
        """
    }
}

stage('Security Scan - Trivy') {
    steps {
        sh """
        trivy image rkg1983/product-catalog:${BUILD_NUMBER}
        """
    }
}

stage('Docker Push') {
    steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
            sh """
            docker login -u $USER -p $PASS
            docker push rkg1983/product-catalog:${BUILD_NUMBER}
            """
        }
    }
}

        stage('Update Kubernetes Manifest') {
            steps {
                withCredentials([string(credentialsId: 'github-token', variable: 'TOKEN')]) {
                    sh '''
                    # Ensure branch is up-to-date
                    git fetch origin cicd
                    git checkout cicd
                    git reset --hard origin/cicd

                    # Update Kubernetes manifest safely
                    sed -i "s|image: .*|image: $DOCKER_USERNAME/product-catalog:$BUILD_NUMBER|" kubernetes/productcatalog/deploy.yaml

                    # Configure Git for CI
                    git config user.email "ravikant@gmail.com"
                    git config user.name "Ravikant Gupta"

                    # Commit and push only if changes exist
                    if ! git diff --quiet; then
                        git add kubernetes/productcatalog/deploy.yaml
                        git commit -m "[CI]: Update product catalog image tag $BUILD_NUMBER"
                        git push https://$TOKEN@github.com/ravikant1983/ultimate-devops-project-demo.git HEAD:cicd
                    else
                        echo "No changes to commit"
                    fi
                    '''
                }
            }
        }

        stage('Trigger ArgoCD Deployment') {
            steps {
		withCredentials([string(credentialsId: 'argocd-token', variable: 'ARGOCD_TOKEN')]) {
                    sh """
		    argocd login a853d1ab52bbf48a19db1763162e88a1-447774326.ap-south-1.elb.amazonaws.com \
                    --grpc-web \
		    --insecure \
                    --auth-token $ARGOCD_TOKEN

		    argocd app sync product-catalog-app \
		    --grpc-web \
		    --insecure
                    """
            }
        }

    }

}
}
