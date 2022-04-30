pipeline {
        environment {
          imagename = "shubhradeepghosh23/test-app"
          tag = '1.0.4'
          docker_host = '172.31.90.253'
          registryCredential = 'dockerhub-cred'
          dockerImage = ''
          CHECK_URL = "http://52.55.157.249:8085/greeting"
          CMD = "curl --write-out %{http_code} --silent --output /dev/null ${CHECK_URL}"
    }
        agent any
        tools {
            jdk 'jdk11'
            maven 'maven3'
    }
        stages {
          stage ('Initialize') {
            steps {
                sh '''
                    echo "PATH = ${PATH}"
                    echo "M2_HOME = ${M2_HOME}"
                '''
            }
        }
   }

          stage ('Build') {
            steps {
                sh 'mvn clean -Dmaven.test.failure.ignore=true install' 
            }
            post {
                success {
                    junit 'target/surefire-reports/**/*.xml' 
                }
            }
        }

          stage ("SonarQube analysis") {
            agent any
            steps {
              withSonarQubeEnv('sonar') {
                sh 'mvn clean package sonar:sonar'
              }
            }
          }
          stage("Quality Gate") {
            steps {
              timeout(time: 1, unit: 'HOURS') {
                waitForQualityGate abortPipeline: true
              }
            }
          }
          stage('Building Docker Image') {
            steps{
              script {
                dockerImage = docker.build imagename
              }
            }
          }
          stage('Push Image to Docker Registry') {
            steps{
              script {
                docker.withRegistry( 'https://registry.hub.docker.com', registryCredential ) {
                dockerImage.push("$BUILD_NUMBER")
                dockerImage.push(tag)

              }
            }
          }
        }
          
          stage('Deploy to Docker Container on Remote Docker Host EC2') {
            steps{   
              sshagent(credentials: ['docker-host-id']) {
                sh "ssh -o StrictHostKeyChecking=no ubuntu@${docker_host} uptime"
                sh "sudo docker run -d -p 8085:8085 ${imagename}:${tag}"
                sleep 30 
              }
            }
          } 


           stage('Health Check Stage-One') {
             steps {
               script{
                 sh "${CMD} > commandResult"
                 env.status = readFile('commandResult').trim()
                }
            }
        }
           stage('Health Check Stage-Two') {
             steps {
               script {
                 sh "echo ${env.status}"
                   if (env.status == '200') {
                     currentBuild.result = "SUCCESS"
                  }
                   else {
                     currentBuild.result = "FAILURE"
                  }
              }
          }
      }

      post {
        always {
          emailext body: '$PROJECT_NAME - Build # $BUILD_NUMBER - $BUILD_STATUS! \n Check console output at $BUILD_URL to view the results.', recipientProviders: [[$class: 'DevelopersRecipientProvider'], [$class: 'RequesterRecipientProvider']], subject: '$PROJECT_NAME - Build # $BUILD_NUMBER - $BUILD_STATUS!!'
          cleanWs()
        }
      }
}

      