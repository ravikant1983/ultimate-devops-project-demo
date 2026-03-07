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
                sh """
                sed -i "s|image: .*|image: $IMAGE_NAME:$IMAGE_TAG|" kubernetes/productcatalog/deploy.yaml
                git config --global user.email "ravikant@gmail.com"
                git config --global user.name "Ravikant Gupta"
                git add kubernetes/productcatalog/deploy.yaml
                git commit -m "[CI] Update image tag to $IMAGE_TAG"
                git push https://$GITHUB_TOKEN@github.com/ravikant1983/ultimate-devops-project-demo.git HEAD:main
                """
            }
        }

        stage('Trigger ArgoCD Deployment') {
            steps {
                sh """
                argocd app sync product-catalog-app --grpc-web --auth-token argocd-token
                """
            }
        }

    }

    post {
        success {
            echo "Pipeline completed successfully!"
        }
        failure {
            slackSend channel: '#devops', message: "Build #${BUILD_NUMBER} failed! Check Jenkins logs."
        }
    }
}
