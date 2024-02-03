pipeline {
  agent any

  stages {
      stage('Build Artifact - Maven') {
            steps {
              sh "mvn clean package -DskipTests=true" 
              archive 'target/*.jar' //so that they can be downloaded later
            }
      }

      stage('Unit Tests - JUnit and JaCoCo') {
            steps {
              sh "mvn test"
            }
            post {
              always {
                junit 'target/surefire-reports/*.xml'
                jacoco execPattern: 'target/jacoco.exec'
              }
            }
      }

      stage ('Docker Build and Push'){
            steps{
                withDockerRegistry([credentialsId: 'docker-hub', url: ""]) {
                    sh "printenv"
                    sh "docker build -t drugman21/numeric-app ."
                    sh "docker push drugman21/numeric-app:latest"
                }      
            }
      }
  }
}