pipeline {
  agent any

  environment {
    DOCKER_USER = "ahsam123"
    IMAGE_NAME  = "spring-demo"
    SONAR_HOST  = "http://203.124.45.203:9000"
    NEXUS_URL   = "http://203.124.45.203:8081"
  }

  stages {

    stage("Checkout") {
      steps {
        checkout scm
      }
    }

    stage("Build & Test (Maven)") {
      steps {
        sh "./mvnw clean test package"
      }
    }
stage("Creds Check") {
  steps {
    echo "Checking sonar + DockerHub + nexus credential visibility..."
    withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
      sh 'echo "SONAR credential OK"'
    }
    withCredentials([usernamePassword(credentialsId: 'DockerHub', usernameVariable: 'U', passwordVariable: 'P')]) {
      sh 'echo "DockerHub credential OK"'
    }
    withCredentials([usernamePassword(credentialsId: 'nexus', usernameVariable: 'U', passwordVariable: 'P')]) {
      sh 'echo "Nexus credential OK"'
    }
  }
}

    stage("SonarQube Analysis") {
      steps {
        withCredentials([string(credentialsId: 'sonar', variable: 'SONAR_TOKEN')]) {
          sh """
          ./mvnw sonar:sonar \
            -Dsonar.projectKey=spring-demo \
            -Dsonar.host.url=${SONAR_HOST} \
            -Dsonar.login=${SONAR_TOKEN}
          """
        }
      }
    }

    stage("Upload Artifact to Nexus") {
      steps {
        sh "./mvnw -s /var/jenkins_home/.m2/settings.xml deploy -DskipTests=true"
      }
    }

    stage("Docker Build") {
      steps {
        sh """
        docker build -t ${DOCKER_USER}/${IMAGE_NAME}:${BUILD_NUMBER} .
        docker tag ${DOCKER_USER}/${IMAGE_NAME}:${BUILD_NUMBER} ${DOCKER_USER}/${IMAGE_NAME}:latest
        """
      }
    }

    stage("Trivy Image Scan") {
      steps {
        sh """
        trivy image --severity HIGH,CRITICAL --exit-code 1 \
          ${DOCKER_USER}/${IMAGE_NAME}:${BUILD_NUMBER}
        """
      }
    }

    stage("Push Image") {
      steps {
        withCredentials([usernamePassword(credentialsId: 'DockerHub', usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
          sh """
          echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
          docker push ${DOCKER_USER}/${IMAGE_NAME}:${BUILD_NUMBER}
          docker push ${DOCKER_USER}/${IMAGE_NAME}:latest
          """
        }
      }
    }

    stage("Deploy to Kubernetes") {
      steps {
        sh """
        sed -i 's|DOCKER_USER|${DOCKER_USER}|g' k8s/deployment.yaml
        sed -i 's|TAG|${BUILD_NUMBER}|g' k8s/deployment.yaml

        kubectl apply -f k8s/deployment.yaml
        kubectl apply -f k8s/service.yaml

        kubectl rollout status deployment/spring-demo
        """
      }
    }

    stage("Verify Deployment") {
      steps {
        sh """
          kubectl get pods -l app=spring-demo
          kubectl get svc spring-demo-svc
          kubectl get endpoints spring-demo-svc
        """
      }
    }
  }
}
