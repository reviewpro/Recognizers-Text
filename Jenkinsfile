@Library("pipeline-utils@master")
import com.reviewpro.*

gitPipelineUtils = new gitUtils()
mavenPipelineUtils = new mavenUtils()
tagPipelineUtils = new tagUtils()
rundeckPipelineUtils = new rundeckUtils()
notifierPipelineUtils = new notifierUtils()
dependencyCheckPipelineUtils = new dependencyCheckUtils()
sonarOwasp = new sonarqubeUtils()

def nextTag = ''

pipeline {
  agent any

  environment {
    CODEARTIFACT_AUTH_TOKEN = sh(script: "aws codeartifact get-authorization-token --domain reviewpro --domain-owner 864066779100 --region eu-central-1 --query authorizationToken --output text", returnStdout: true).trim()
  }

  options {
    disableConcurrentBuilds()
    skipDefaultCheckout()
    skipStagesAfterUnstable()
  }

  stages {

    stage ('Checkout') {
      steps {
        checkout scm
        script {
          gitPipelineUtils.gitCheckout(env.BRANCH_NAME)
          nextTag = tagPipelineUtils.getNextReleaseTag()
        }
      }
    }

    stage ('Build') {
      steps {
        dir ("Java") {
          sh 'mvn -U clean compile'
        }
      }
    }

    stage ('Publish') {
      when {
        branch 'master'
      }
      steps {
        dir ("Java") {
          script {
            mavenPipelineUtils.publish(nextTag)
          }
        }
      }
    }

    stage ('Hotfix') {
      when {
        allOf {
          branch "hotfix-*"
          expression {
            env.BRANCH_NAME ==~ '^hotfix-[0-9]+\\.[0-9]+\\.[0-9]+\$'
          }
        }
      }
      steps {
        dir ("Java") {
          script {
            mavenPipelineUtils.publishHotfix(env.BRANCH_NAME)
          }
        }
      }
    }
  }

  post {
    success {
      script {
        if (currentBuild.previousBuild != null && currentBuild.previousBuild.result != 'SUCCESS') {
          notifierPipelineUtils.notifySuccess(currentBuild, env.BUILD_URL, "pd@reviewpro.com", nextTag)
        }
        if (params.CLOSE) {
          notifierPipelineUtils.notifyClosing(currentBuild, env.BUILD_URL, "pd@reviewpro.com", nextTag)
        }
        deleteDir()
      }
    }
    failure {
      script {
        notifierPipelineUtils.notifyError(currentBuild, env.BUILD_URL, "pd@reviewpro.com", nextTag)
      }
    }
  }

}
