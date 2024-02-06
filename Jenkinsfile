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

      stage('Mutation Tests - PIT') {
      steps {
        sh "mvn org.pitest:pitest-maven:mutationCoverage"
      }
      post {
        always {
          pitmutation mutationStatsFile: '**/target/pit-reports/**/mutations.xml'
        }
      }
    }

      stage('SonarQube - SAST') {
         steps {
           withSonarQubeEnv('SonarQube') {
             sh "mvn clean verify sonar:sonar -Dsonar.projectKey=numeric-application -Dsonar.projectName='numeric-application' -Dsonar.host.url=http://devsecops-demo.francecentral.cloudapp.azure.com:9000 -Dsonar.token=sqp_7f4529e9c105380f3d0f622ef706f7c89e480617"
           }
           timeout(time: 2, unit: 'MINUTES') {
             script {
               waitForQualityGate abortPipeline: true
             }
           }
         }
       }

        stage('Vulnerability Scan - Docker') {

          steps{
                dependencyCheck additionalArguments: '--scan ./ --format XML ', odcInstallation: 'DP-Check'
          }
                   post{
                    always{
                      dependencyCheckPublisher pattern: 'target/dependency-check-report.xml'
                    }
                   }
        }
          // steps {
          //   sh "mvn dependency-check:check"
          // }
          // post{
          //   always{
          //     dependencyCheckPublisher pattern: 'target/dependency-check-report.xml'
          //   }
          // }
        

      stage ('Docker Build and Push'){
            steps{
                withDockerRegistry([credentialsId: 'docker-hub', url: ""]) {
                    sh "printenv"
                    sh "docker build -t drugman21/numeric-app ."
                    sh "docker push drugman21/numeric-app:latest"
                }      
            }
      }

      stage('Kubernetes deployment') {
            steps {
                withKubeConfig([credentialsId: 'kubeconfig']) {
                    sh 'kubectl apply -f k8s_deployment_service.yaml'
                }          
            }
      }
  }
}