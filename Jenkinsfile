pipeline {
  agent any

  environment {
    APP_IMAGE = 'mi-app:latest'
    MAVEN_IMAGE = 'maven:3.9.9-eclipse-temurin-17'
    SONAR_PROJECT_KEY = 'mi-app'
    SONAR_HOST_URL = 'http://sonarqube:9000'
    JENKINS_VOLUME = 'jenkins_home'
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Test') {
      steps {
        sh '''
          docker run --rm \
            --network cicd-net \
            -v "$JENKINS_VOLUME:/var/jenkins_home" \
            -w "$WORKSPACE" \
            "$MAVEN_IMAGE" \
            mvn clean test \
              -Dgroups=au.com.equifax.cicddemo.domain.UnitTest,au.com.equifax.cicddemo.domain.IntegrationTest
        '''
      }
    }

    stage('Build') {
      steps {
        sh '''
          docker run --rm \
            --network cicd-net \
            -v "$JENKINS_VOLUME:/var/jenkins_home" \
            -w "$WORKSPACE" \
            "$MAVEN_IMAGE" \
            mvn -DskipTests package
        '''
      }
    }

    stage('Docker Build') {
      steps {
        sh 'docker build -t "$APP_IMAGE" .'
      }
    }

    stage('Static Analysis (SonarQube)') {
      steps {
        withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
          sh '''
            docker run --rm \
              --user root \
              --network cicd-net \
              -e SONAR_HOST_URL="$SONAR_HOST_URL" \
              -e SONAR_TOKEN="$SONAR_TOKEN" \
              -v "$JENKINS_VOLUME:/var/jenkins_home" \
              -w "$WORKSPACE" \
              sonarsource/sonar-scanner-cli \
              -Dsonar.scanner.metadataFilePath="$WORKSPACE/report-task.txt"
          '''
        }
      }
    }

    stage('Quality Gate') {
      steps {
        withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
          sh '''
            set -eu

            TASK_URL=$(grep '^ceTaskUrl=' report-task.txt | cut -d= -f2-)

            while true; do
              RESPONSE=$(curl -s -u "$SONAR_TOKEN:" "$TASK_URL")
              STATUS=$(echo "$RESPONSE" | sed -n 's/.*"status":"\\([^"]*\\)".*/\\1/p')

              if [ "$STATUS" = "SUCCESS" ]; then
                ANALYSIS_ID=$(echo "$RESPONSE" | sed -n 's/.*"analysisId":"\\([^"]*\\)".*/\\1/p')
                break
              fi

              if [ "$STATUS" = "FAILED" ]; then
                echo "SonarQube background task failed"
                exit 1
              fi

              sleep 5
            done

            QG_STATUS=$(curl -s -u "$SONAR_TOKEN:" "$SONAR_HOST_URL/api/qualitygates/project_status?analysisId=$ANALYSIS_ID" | sed -n 's/.*"status":"\\([^"]*\\)".*/\\1/p')
            HOTSPOTS=$(curl -s -u "$SONAR_TOKEN:" "$SONAR_HOST_URL/api/hotspots/search?projectKey=$SONAR_PROJECT_KEY&status=TO_REVIEW&p=1&ps=1" | sed -n 's/.*"total":\\([0-9][0-9]*\\).*/\\1/p' | head -n 1)

            [ -n "$HOTSPOTS" ] || HOTSPOTS=0

            echo "Quality gate: $QG_STATUS"
            echo "Security hotspots pending review: $HOTSPOTS"

            [ "$QG_STATUS" = "OK" ] || exit 1
            [ "$HOTSPOTS" -eq 0 ] || exit 1
          '''
        }
      }
    }

    stage('Container Security Scan (Trivy)') {
      steps {
        sh 'trivy image --severity CRITICAL --exit-code 1 --no-progress "$APP_IMAGE"'
      }
    }

    stage('Deploy') {
      when {
        anyOf {
          branch 'main'
          branch 'master'
        }
      }
      steps {
        sh '''
          docker rm -f mi-app || true
          docker run -d --name mi-app --network cicd-net -p 8081:8080 "$APP_IMAGE"
        '''
      }
    }
  }

  post {
    failure {
      echo 'Pipeline failed. Review Jenkins console, SonarQube, and Trivy output.'
    }
    always {
      junit allowEmptyResults: true, testResults: 'target/surefire-reports/*.xml'
      cleanWs()
    }
  }
}
