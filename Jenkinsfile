import groovy.json.JsonOutput

def slackSendNotification(color = '', messageType = '', nonGenericMessage = false) {
  if (color != '' && messageType != '') {
    // slackSend
    withCredentials([
      [$class: 'StringBinding', credentialsId: 'SLACK_INTEGRATION_TOKEN', variable: 'SLACK_INTEGRATION_TOKEN'],
      string(credentialsId: 'devops-slack-channel', variable: 'SLACK_CHANNEL')
    ]) {
      if (nonGenericMessage) {
        message = messageType
      } else {
        message = "*${messageType}*: Job ${env.JOB_NAME} build ${env.BUILD_NUMBER}"
      }
      message += " (<${env.BUILD_URL}|Open>)"
      def payload = JsonOutput.toJson([
        "channel": "${env.SLACK_CHANNEL}",
        "attachments": [[
          "fallback": "${message}",
          "color": "${color}",
          "fields": [[
            "short": false,
            "value": "${message}",
          ]],
          "mrkdwn_in": [
            "pretext",
            "text",
            "fields",
          ]
        ]]
      ])
      apiUrl = "https://${env.SLACK_TEAM_DOMAIN}.slack.com/services/hooks/jenkins-ci?token=${env.SLACK_INTEGRATION_TOKEN}"
      response = sh(returnStdout: true,
        script: "curl -s -H \"Content-type: application/json\" -X POST -d '${payload}' ${apiUrl}"
      ).trim()
    }
  }
}

pipeline {
  agent {
    node { label "${WORKER}" }
  }
  environment {
    registryCredential = 'jenkins-artifactory'
  }
  stages {
    stage('ShellCheck') {
      when {
        not { tag pattern: "^v\\d+\\.\\d+\\.\\d+\$", comparator: "REGEXP" }
      }
      steps {
        script {
          sh "cd configurator && make shellcheck"
        }
      }
    }
    stage('Test') {
      when {
        not { tag pattern: "^v\\d+\\.\\d+\\.\\d+\$", comparator: "REGEXP" }
      }
      steps{
        withCredentials([string(credentialsId: 'ARTIFACTORY_URL', variable: 'ARTIFACTORY_URL')]) {
          script {
              sh(
                "cd configurator && " +
                "IMAGE_NAME=${env.ARTIFACTORY_URL}/configurator:${env.BUILD_NUMBER} make test"
              )
          }
        }
      }
      post {
        cleanup {
          withCredentials([string(credentialsId: 'ARTIFACTORY_URL', variable: 'ARTIFACTORY_URL')]) {
            script {
              sh("docker rmi ${env.ARTIFACTORY_URL}/configurator:${env.TAG_NAME} || /bin/true")
            }
          }
        }
      }
    }
    stage('Test uber_tar') {
      when {
        not { tag pattern: "^v\\d+\\.\\d+\\.\\d+\$", comparator: "REGEXP" }
      }
      steps{
        script {
          docker.withRegistry("https://quay.io", "QUAY") {
            sh(
              "cd configurator && " +
              "make test_uber_tar"
            )
          }
        }
      }
      post {
        cleanup {
          script {
            sh "cd configurator && make clean"
            sh "docker rmi -f \$(docker images -qa) || /bin/true"
          }
        }
      }
    }
    stage('Push internal image') {
      when {
        tag pattern: "^v\\d+\\.\\d+\\.\\d+-rc\\d+\$", comparator: "REGEXP"
      }
      steps{
        withCredentials([string(credentialsId: 'ARTIFACTORY_URL', variable: 'ARTIFACTORY_URL')]) {
          script {
              nonRCTag = env.TAG_NAME.replaceAll(/-rc\d+/, '')
              dockerRCImage = "${env.ARTIFACTORY_URL}/configurator:${env.TAG_NAME}"
              dockerNonRCImage = "${env.ARTIFACTORY_URL}/configurator:${nonRCTag}"
              docker.withRegistry("https://${env.ARTIFACTORY_URL}", registryCredential) {
                sh(
                  "cd configurator && IMAGE_NAME=${dockerRCImage} make push && " +
                  "docker tag ${dockerRCImage} ${dockerNonRCImage} && " +
                  "docker push ${dockerNonRCImage}"
                )
              }
          }
        }
      }
      post {
        success {
        withCredentials([string(credentialsId: 'ARTIFACTORY_URL', variable: 'ARTIFACTORY_URL')]) {
            script {
              slackSendNotification("${env.SLACK_COLOR_GOOD}", "Pushed docker image: ${env.ARTIFACTORY_URL}/configurator:${env.TAG_NAME}")
            }
          }
        }
        cleanup {
        withCredentials([string(credentialsId: 'ARTIFACTORY_URL', variable: 'ARTIFACTORY_URL')]) {
            script {
              sh("docker rmi ${env.ARTIFACTORY_URL}/configurator:${env.TAG_NAME} || /bin/true")
            }
          }
        }
      }
    }
    stage('Push internal uber_image') {
      when {
        tag pattern: "^v\\d+\\.\\d+\\.\\d+-rc\\d+\$", comparator: "REGEXP"
      }
      steps{
        withCredentials([string(credentialsId: 'ARTIFACTORY_URL', variable: 'ARTIFACTORY_URL')]) {
          script {
              nonRCTag = env.TAG_NAME.replaceAll(/-rc\d+/, '')
              dockerImage = "${env.ARTIFACTORY_URL}/configurator:${env.TAG_NAME}"
              uberImage = "${env.ARTIFACTORY_URL}/configurator:uber-${env.TAG_NAME}"
              uberNonRCImage = "${env.ARTIFACTORY_URL}/configurator:uber-${nonRCTag}"
              docker.withRegistry("https://${env.ARTIFACTORY_URL}", registryCredential) {
                sh(
                  "cd configurator && IMAGE_NAME=${dockerImage} UBER_IMAGE_NAME=${uberImage} make push_uber_tar && " +
                  "docker tag ${uberImage} ${uberNonRCImage} && " +
                  "docker push ${uberNonRCImage}"
                )
              }
          }
        }
      }
      post {
        success {
        withCredentials([string(credentialsId: 'ARTIFACTORY_URL', variable: 'ARTIFACTORY_URL')]) {
            script {
              slackSendNotification("${env.SLACK_COLOR_GOOD}", "Pushed docker image: ${env.ARTIFACTORY_URL}/configurator:uber-${env.BUILD_NUMBER}")
            }
          }
        }
        cleanup {
        withCredentials([string(credentialsId: 'ARTIFACTORY_URL', variable: 'ARTIFACTORY_URL')]) {
            script {
              sh("docker rmi ${env.ARTIFACTORY_URL}/configurator:${env.TAG_NAME} || /bin/true")
              sh("docker rmi ${env.ARTIFACTORY_URL}/configurator:uber-${env.TAG_NAME} || /bin/true")
            }
          }
        }
      }
    }
    stage('Promote image to quay') {
      when {
        tag pattern: "^v\\d+\\.\\d+\\.\\d+\$", comparator: "REGEXP"
      }
      steps{
        withCredentials([string(credentialsId: 'ARTIFACTORY_URL', variable: 'ARTIFACTORY_URL')]) {
          script {
              dockerImage = "${env.ARTIFACTORY_URL}/configurator:${env.TAG_NAME}"
              docker.withRegistry("https://${env.ARTIFACTORY_URL}", registryCredential) {
                sh("docker pull ${dockerImage}")
              }
              docker.withRegistry("https://quay.io", "QUAY") {
                sh(
                  "docker tag ${dockerImage} quay.io/sysdig/configurator:${env.TAG_NAME} && " +
                  "docker push quay.io/sysdig/configurator:${env.TAG_NAME}"
                )
              }
          }
        }
      }
      post {
        success {
          script {
            slackSendNotification("${env.SLACK_COLOR_GOOD}", "Pushed docker image: quay.io/sysdig/configurator:${env.TAG_NAME}")
          }
        }
        cleanup {
        withCredentials([string(credentialsId: 'ARTIFACTORY_URL', variable: 'ARTIFACTORY_URL')]) {
            script {
              sh("docker rmi ${env.ARTIFACTORY_URL}/configurator:${env.TAG_NAME} || /bin/true")
            }
          }
        }
      }
    }
    stage('Promote uber_image to quay') {
      when {
        tag pattern: "^v\\d+\\.\\d+\\.\\d+\$", comparator: "REGEXP"
      }
      steps{
        withCredentials([string(credentialsId: 'ARTIFACTORY_URL', variable: 'ARTIFACTORY_URL')]) {
          script {
              uberImage = "${env.ARTIFACTORY_URL}/configurator:uber-${env.TAG_NAME}"
              docker.withRegistry("https://${env.ARTIFACTORY_URL}", registryCredential) {
                sh("docker pull ${uberImage}")
              }
              docker.withRegistry("https://quay.io", "QUAY") {
                "docker tag ${uberImage} quay.io/sysdig/configurator:uber-${env.TAG} && " +
                "docker push quay.io/sysdig/configurator:${env.TAG_NAME}"
              }
          }
        }
      }
      post {
        success {
          script {
            slackSendNotification("${env.SLACK_COLOR_GOOD}", "Pushed docker image: quay.io/sysdig/configurator:uber-${env.TAG_NAME}")
          }
        }
        cleanup {
        withCredentials([string(credentialsId: 'ARTIFACTORY_URL', variable: 'ARTIFACTORY_URL')]) {
            script {
              sh("docker rmi ${env.ARTIFACTORY_URL}/configurator:uber-${env.TAG_NAME} || /bin/true")
            }
          }
        }
      }
    }
  }
  post {
    failure {
      script {
        slackSendNotification("${env.SLACK_COLOR_DANGER}", "FAILED")
      }
    }
    success {
      script {
        slackSendNotification("${env.SLACK_COLOR_GOOD}", "SUCCESS")
      }
    }
  }
}