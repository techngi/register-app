pipeline {
  agent { label 'Jenkins-Agent' }
  tools {
    jdk 'java17'
    maven 'maven3'
  }
  environment {
    APP_NAME = "register-app-pipeline"
    RELEASE = "1.0.0"
    IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
    JENKINS_API_TOKEN = credentials("JENKINS_API_TOKEN")
  }

  stages {
    stage("Cleanup Workspace") {
      steps {
        cleanWs()
      }
    }

    stage("Checkout from SCM") {
      steps {
        git branch: 'main', credentialsId: 'github', url: 'https://github.com/techngi/register-app'
      }
    }

    stage("Build Application") {
      steps {
        sh "mvn clean package"
      }
    }

    stage("Test Application") {
      steps {
        sh "mvn test"
      }
    }

    stage("SonarQube Analysis") {
      steps {
        script {
          withSonarQubeEnv(credentialsId: 'jenkins-sonarqube-token') {
            sh "mvn sonar:sonar"
          }
        }
      }
    }

    stage("Quality Gate") {
      steps {
        script {
          waitForQualityGate abortPipeline: false, credentialsId: 'jenkins-sonarqube-token'
        }
      }
    }

    stage("Build & Push Docker Image") {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          script {
            def imageName = "${DOCKER_USER}/${APP_NAME}"
            docker.withRegistry('https://index.docker.io/v1/', 'dockerhub') {
              def docker_image = docker.build(imageName)
              docker_image.push("${IMAGE_TAG}")
              docker_image.push('latest')
            }
          }
        }
      }
    }

    stage("Trivy Scan") {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          script {
            def imageName = "${DOCKER_USER}/${APP_NAME}"
            sh "docker run -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image ${imageName}:latest --no-progress --scanners vuln --exit-code 0 --severity HIGH,CRITICAL --format table"
          }
        }
      }
    }

    stage("Cleanup Artifacts") {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
          script {
            def imageName = "${DOCKER_USER}/${APP_NAME}"
            sh "docker rmi ${imageName}:${IMAGE_TAG}"
            sh "docker rmi ${imageName}:latest"
          }
        }
      }
    }

    stage("Trigger CD Pipeline") {
      steps {
        script {
          sh "curl -v -k --user clouduser:${JENKINS_API_TOKEN} -X POST -H 'cache-control: no-cache' -H 'content-type: application/x-www-form-urlencoded' --data 'IMAGE_TAG=${IMAGE_TAG}' 'http://ec2-13-232-128-192.ap-south-1.compute.amazonaws.com:8080/job/gitops-register-app-cd/buildWithParameters?token=gitops-token'"
        }
      }
    }
  }

  post {
    failure {
      emailext body: '''${SCRIPT, template="groovy-html.template"}''',
               subject: "${env.JOB_NAME} - Build # ${env.BUILD_NUMBER} - Failed",
               mimeType: 'text/html',
               to: "sanaqvi573@gmail.com"
    }
    success {
      emailext body: '''${SCRIPT, template="groovy-html.template"}''',
               subject: "${env.JOB_NAME} - Build # ${env.BUILD_NUMBER} - Successful",
               mimeType: 'text/html',
               to: "sanaqvi573@gmail.com"
    }
  }
}
