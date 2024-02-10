pipeline {
  agent any


  environment {
    deploymentName = "devsecops"
    containerName = "devsecops-container"
    serviceName = "devsecops-svc"
    imageName = "drugman21/numeric-app"
    applicationURL="devsecops-demo.francecentral.cloudapp.azure.com"
    applicationURI="/increment/99"
  }

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
          steps {
            parallel(
                "Dependency Scan": {
                        sh "mvn dependency-check:check"
                 },
                 "Trivy Scan":{
                        sh "bash trivy-docker-image-scan.sh"
                 },
                 "OPA Conftest":{
	 			                sh 'docker run --rm -v $(pwd):/project openpolicyagent/conftest test --policy opa-docker-security.rego Dockerfile'
                 }
            )
          }
        }
          // steps{
          //       dependencyCheck additionalArguments: '--scan ./ --format XML ', odcInstallation: 'DP-Check'
          //       dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
          //   }
        

          // steps{
          //       dependencyCheck additionalArguments: '--scan ./ --format XML ', odcInstallation: 'DP-Check'
          // }
          //          post{
          //           always{
          //             dependencyCheckPublisher pattern: 'target/dependency-check-report.xml'
          //           }
          //          }
        
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
                    sh "sudo docker build -t drugman21/numeric-app ."
                    sh "docker push drugman21/numeric-app:latest"
                }      
            }
      }

      // stage('Vulnerability Scan - Kubernetes') {
      //     steps {
      //       sh 'docker run --rm -v $(pwd):/project openpolicyagent/conftest test --policy opa-k8s-security.rego k8s_deployment_service.yaml'
      //   }
      // }

      stage('Vulnerability Scan - Kubernetes') {
          steps {
            parallel(
              "OPA Scan ": {
                sh 'docker run --rm -v $(pwd):/project openpolicyagent/conftest test --policy opa-k8s-security.rego k8s_deployment_service.yaml'
              },
              "Kubesec Scan": {
                sh "bash kubesec-scan.sh"
              }
            )
        }
      }


      // stage('Kubernetes Deployment - DEV') {
      //       steps {
      //         parallel(
      //           "Deployment": {
      //             withKubeConfig([credentialsId: 'kubeconfig']) {
      //               sh 'bash k8s-deployment.sh'
      //             }  
      //           },
      //           "Rollout status": {
      //             withKubeConfig([credentialsId: 'kubeconfig']) {
      //               sh 'bash k8s-deployment-rollout-status.sh'
      //             }  
      //           }
      //         )                      
      //       }
      // }


  }

  post{
    always{
      junit 'target/surefire-reports/*.xml'
      jacoco execPattern: 'target/jacoco.exec'
      //pitmutation mutationStatsFile: '**/target/pit-reports/**/mutations.xml'
      dependencyCheckPublisher pattern: 'target/dependency-check-report.xml'
    }
  }
}