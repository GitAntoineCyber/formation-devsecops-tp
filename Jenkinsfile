@Library('Slack') _
pipeline {
  agent any   
	environment {
        SLACK_CHANNEL = 'teamDevsecops' // Slack channel to send notifications
    }

  stages {
      stage('Build Artifact') {
            steps {
              sh "mvn clean package -DskipTests=true"
              archive 'target/*.jar' //so that they can be downloaded later
            }
        }  
     //--------------------------
    stage('UNIT test & jacoco ') {
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
//--------------------------
    stage('Mutation Tests - PIT') {
      steps {
        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
          sh "mvn org.pitest:pitest-maven:mutationCoverage"
        }
      }
        post { 
         always { 
           pitmutation mutationStatsFile: '**/target/pit-reports/**/mutations.xml'
         }
       }
    }
    //--------------------------
    stage('Analyse Sonarqube') {
      steps {
        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
          withCredentials([string(credentialsId: 'sonarqube-token', variable: 'TOKENSONAR')]) {
          sh "mvn sonar:sonar \
  -Dsonar.projectKey=Cyber\
  -Dsonar.host.url=http://devsecops69.eastus.cloudapp.azure.com:9999 \
  -Dsonar.login=${TOKENSONAR}"
          }
          
        }
      }
  
    }
//--------------------------
    stage('Vulnerability Scan owasp - dependency-check') {
   steps {
	    catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
     		sh "mvn dependency-check:check"
	    }
		}
 	post { 
         always { 
           dependencyCheckPublisher pattern: 'target/dependency-check-report.xml'
         }
       }
}
//--------------------------
//--------------------------
    stage('Docker Build and Push') {
      steps {
        withCredentials([string(credentialsId: '001', variable: 'DOCKER_HUB_PASSWORD')]) {
          sh 'sudo docker login -u antoinecyberdocker -p $DOCKER_HUB_PASSWORD'
          sh 'printenv'
          sh 'sudo docker build -t antoinecyberdocker/devops-app:""$GIT_COMMIT"" .'
          sh 'sudo docker push antoinecyberdocker/devops-app:""$GIT_COMMIT""'
        }
 
      }
    }
//--------------------------  
	  stage('Vulnerability Scan - Docker Trivy') {
       steps {
	        withCredentials([string(credentialsId: 'trivy_github_token', variable: 'TOKEN')]) {
			 catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                 sh "sed -i 's#token_github#${TOKEN}#g' trivy-image-scan.sh"
                 sh "sudo bash trivy-image-scan.sh"
		}
	   }
    	}
	       
}
//--------------------------
	  stage('Vulnerability Scan - Kubernetes') {
      steps {
        parallel(
          "OPA Scan": {
            sh 'sudo docker run --rm -v /home/devsecops/formation-devsecops-tp:/project openpolicyagent/conftest test --policy opa-k8s-security.rego k8s_deployment_service.yaml'
          },
          "Kubesec Scan": {
            sh "sudo bash kubesec-scan.sh"
          },
          "Trivy Scan": {
            sh "sudo bash trivy-k8s-scan.sh"
          }

        )
      }
    }
//--------------------------
	    stage('Deployment Kubernetes  ') {
      steps {
        withKubeConfig([credentialsId: '002']) {
              sh "sed -i 's#replace#antoinecyberdocker/devops-app:${GIT_COMMIT}#g' k8s_deployment_service.yaml"
              sh 'kubectl apply -f k8s_deployment_service.yaml'
        }
      }
    }
//--------------------------
	 stage('Scan API with OWASP ZAP') {
            steps {
		 catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                // Exécutez le script de scan
                sh 'bash ./zap.sh' 
		archive 'owasp-zap-report/*.html'
}
            }
        }
//--------------------------
        stage('Report Handling') {
            steps {
                // Gérer les rapports générés
                script {
                    def exitCode = sh(script: 'cat owasp-zap-report/zap_report.html', returnStatus: true)
                    if (exitCode != 0) {
                        echo "OWASP ZAP Report has either Low/Medium/High Risk. Please check the HTML Report"
                        currentBuild.result = 'FAILURE'
                    } else {
                        echo "OWASP ZAP did not report any Risk"
                    }
                }
            }
        }
  }
	post {
        success {
            script {
                sendNotification('SUCCESS')
            }
        }
        failure {
            script {
                sendNotification('FAILURE')
            }
        }
        unstable {
            script {
                sendNotification('UNSTABLE')
            }
        }
    }
}
//--------------------------


//--------------------------
  
