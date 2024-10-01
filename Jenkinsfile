pipeline {
  agent any

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
          sh "mvn clean verify sonar:sonar \
  -Dsonar.projectKey=CyberOPS \
  -Dsonar.projectName='CyberOPS' \
  -Dsonar.host.url=http://devsecops69.eastus.cloudapp.azure.com:9000 \
  -Dsonar.token=sqp_f031552b22d6cf10fcfb41f19ca15b15aa2fd1fc
"
        }
      }
  
    }
//--------------------------
//--------------------------

    //--------------------------
    }
}
