@Library('slack') _

pipeline {
  agent any


  environment {
    deploymentName = "devsecops"
    containerName = "devsecops-container"
    serviceName = "devsecops-svc"
    imageName = "drugman21/numeric-app"
    applicationURL="http://devsecops-demo.francecentral.cloudapp.azure.com"
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
         

      stage ('Docker Build and Push'){
            steps{
                withDockerRegistry([credentialsId: 'docker-hub', url: ""]) {
                    sh "printenv"
                    sh "sudo docker build -t drugman21/numeric-app ."
                    sh "docker push drugman21/numeric-app:latest"
                }      
            }
      }


      stage('Vulnerability Scan - Kubernetes') {
          steps {
            parallel(
              "OPA Scan ": {
                sh 'docker run --rm -v $(pwd):/project openpolicyagent/conftest test --policy opa-k8s-security.rego k8s_deployment_service.yaml'
              },
              "Kubesec Scan": {
                sh "bash kubesec-scan.sh"
              },
              "Trivy Scan": {
                sh "bash trivy-k8s-scan.sh"
              }
            )
        }
      }


      stage('Kubernetes Deployment - DEV') {
            steps {
              parallel(
                "Deployment": {
                  withKubeConfig([credentialsId: 'kubeconfig']) {
                    sh 'bash k8s-deployment.sh'
                  }  
                },
                "Rollout status": {
                  withKubeConfig([credentialsId: 'kubeconfig']) {
                    sh 'bash k8s-deployment-rollout-status.sh'
                  }  
                }
              )                      
            }
      }

      stage('Integration Tests - DEV') {
        steps {
          script {
            try {
              withKubeConfig([credentialsId: 'kubeconfig']) {
                sh "bash integration-test.sh"
              }
            } catch (e) {
              withKubeConfig([credentialsId: 'kubeconfig']) {
                sh "kubectl -n default rollout undo deploy ${deploymentName}"
              }
              throw e
            }
          }
        }
      }

      stage('OWASP ZAP - DAST') {
            steps {
              withKubeConfig([credentialsId: 'kubeconfig']) {
                sh "bash zap.sh"
              }
            }           
      }

      stage('Prompte to PROD?') {
        steps {
          timeout(time: 2, unit: 'DAYS') {
            input 'Do you want to Approve the Deployment to Production Environment/Namespace?'
          }
        }
      }

      // stage('Testing Slack') {
      //   steps {
      //       sh 'exit 0'
      //   }
      // }


  }

  post{
    always{
      junit 'target/surefire-reports/*.xml'
      jacoco execPattern: 'target/jacoco.exec'
      //pitmutation mutationStatsFile: '**/target/pit-reports/**/mutations.xml'
      dependencyCheckPublisher pattern: 'target/dependency-check-report.xml'
      publishHTML([allowMissing: false, alwaysLinkToLastBuild: true, keepAll: true, reportDir: 'owasp-zap-report', reportFiles: 'zap_report.html', reportName: 'OWASP ZAP HTML Report', reportTitles: 'OWASP ZAP HTML Report'])
      //Use sendNotifications.groovy from shared library and provide current build result as parameter 
      sendNotification currentBuild.result
    }
  }
}