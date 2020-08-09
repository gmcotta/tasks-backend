pipeline {
  agent any
  stages {
    stage ('Build Backend') {
      steps {
        bat 'mvnn clean package -DskipTests=true'
      }
    }

    stage ('Unit Tests') {
      steps {
        bat 'mvn test'
      }
    }

    stage ('Sonar Analysis') {
      environment {
        scannerHome = tool 'SONAR_SCANNER'
      }
      steps {
        withSonarQubeEnv('SONAR_LOCAL') {
          bat "\"${scannerHome}/bin/sonar-scanner\" -e -Dsonar.projectKey=DeployBackend -Dsonar.host.url=http://localhost:9000 -Dsonar.login=68c18eb50ad5e831c98ceca00044475d84f1c6b6 -Dsonar.java.binaries=target -Dsonar.coverage.exclusions=**/.mvn/**,**/src/test/**,**/model/**,**Application.java"
        }
      }
    }

    stage ('Quality Gate') {
      steps {
        sleep(20)
        timeout(time: 1, unit: 'MINUTES') {
          waitForQualityGate abortPipeline: true
        }
      }
    }

    stage ('Deploy Backend') {
      steps {
        deploy adapters: [tomcat8(credentialsId: 'tomcat_login', path: '', url: 'http://localhost:8001/')], contextPath: 'tasks-backend', war: 'target/tasks-backend.war'
      }
    }

    stage ('API Tests') {
      steps {
        dir('api-test') {
          git credentialsId: 'login_github', url: 'https://github.com/gmcotta/tasks-api-test'
          bat 'mvn test'
        }
      }
    }

    stage ('Deploy Frontend') {
      steps {
        dir('frontend') {
          git credentialsId: 'login_github', url: 'https://github.com/gmcotta/tasks-frontend'
          bat 'mvn clean package -DskipTests=true'
          deploy adapters: [tomcat8(credentialsId: 'tomcat_login', path: '', url: 'http://localhost:8001/')], contextPath: 'tasks', war: 'target/tasks.war'
        }
      }
    }

    stage ('Functional Tests') {
      steps {
        dir('functional-test') {
          git credentialsId: 'login_github', url: 'https://github.com/gmcotta/tasks-functional-test'
          bat 'mvn test'
        }
      }
    }

    stage('Deploy Prod') {
      steps {
        bat 'wsl docker-compose build'
        bat 'wsl docker-compose up -d'
      }
    }

    stage('Health Check') {
      steps {
        sleep(40)
        dir('functional-test') {
          bat 'mvn verify -Dskip.surefile.tests'
        }
      }
    }
  }

  post {
    always {
      junit allowEmptyResults: true, testResults: 'target/surefile-reports/*.xml, api-test/target/surefile-reports/*.xml, functional-test/target/surefile-reports/*.xml, functional-test/target/failsafe-reports/*.xml'
    }
    unsuccessful {
      emailext attachLog: true, body: 'See the attached log below.', subject: 'Build $BUILD_NUMBER has failed', to: 'gmcotta34+jenkins@gmail.com'
    }
    fixed {
      emailext attachLog: true, body: 'See the attached log below.', subject: 'Build $BUILD_NUMBER is completed successfully', to: 'gmcotta34+jenkins@gmail.com'
    }
  }
}
