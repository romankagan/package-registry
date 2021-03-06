#!/usr/bin/env groovy

@Library('apm@current') _

pipeline {
  agent { label 'docker && linux && immutable' }
  environment {
    BASE_DIR="src/github.com/elastic/package-registry"
    JOB_GIT_CREDENTIALS = "f6c7695a-671e-4f4f-a331-acdce44ff9ba"
    PIPELINE_LOG_LEVEL='INFO'
    DOCKER_REGISTRY = 'docker.elastic.co'
    DOCKER_REGISTRY_SECRET = 'secret/observability-team/ci/docker-registry/prod'
    DOCKER_IMG = "${env.DOCKER_REGISTRY}/package-registry/package-registry"
  }
  options {
    timeout(time: 1, unit: 'HOURS')
    buildDiscarder(logRotator(numToKeepStr: '20', artifactNumToKeepStr: '20', daysToKeepStr: '30'))
    timestamps()
    ansiColor('xterm')
    disableResume()
    durabilityHint('PERFORMANCE_OPTIMIZED')
    rateLimitBuilds(throttle: [count: 60, durationName: 'hour', userBoost: true])
    quietPeriod(10)
  }
  triggers {
    issueCommentTrigger('(?i).*(?:jenkins\\W+)?run\\W+(?:the\\W+)?tests(?:\\W+please)?.*')
  }
  stages {
    /**
    Checkout the code and stash it, to use it on other stages.
    */
    stage('Checkout') {
      steps {
        deleteDir()
        gitCheckout(basedir: "${BASE_DIR}")
        stash allowEmpty: true, name: 'source', useDefaultExcludes: false
      }
    }
    /**
    Checks formatting / linting.
    */
    stage('Lint') {
      steps {
        deleteDir()
        unstash 'source'
        dir("${BASE_DIR}"){
          insideGo{
            sh(label: 'Checks formatting / linting',script: 'mage -debug check  ')
          }
        }
      }
    }
    /**
    Build the project from code..
    */
    stage('Build') {
      steps {
        deleteDir()
        unstash 'source'
        dir("${BASE_DIR}"){
          insideGo(){
            sh(label: 'Checks formatting / linting',script: 'mage -debug build ')
          }
        }
      }
    }
    /**
    Execute unit tests.
    */
    stage('Test') {
      steps {
        deleteDir()
        unstash 'source'
        dir("${BASE_DIR}"){
          insideGo(){
            sh(label: 'Runs the (unit) tests',script: 'mage -debug test ')
          }
        }
      }
      post {
        always {
          junit(allowEmptyResults: true,
            keepLongStdio: true,
            testResults: "${BASE_DIR}/**/junit-*.xml")
        }
      }
    }
    /**
      Publish Docker images.
    */
    stage('Publish Docker image'){
      environment {
        DOCKER_IMG_TAG = "${env.DOCKER_IMG}:${env.GIT_BASE_COMMIT}"
        DOCKER_IMG_TAG_BRANCH = "${env.DOCKER_IMG}:${env.BRANCH_NAME}"
      }
      steps {
        deleteDir()
        unstash 'source'
        dir("${BASE_DIR}"){
          dockerLogin(secret: "${env.DOCKER_REGISTRY_SECRET}",
            registry: "${env.DOCKER_REGISTRY}")
          sh(label: 'Build Docker image',
            script: "docker build -t ${env.DOCKER_IMG_TAG} .")
          sh(label: 'Push Docker image sha',
            script: "docker push ${env.DOCKER_IMG_TAG}")
          sh(label: 'Re-tag Docker image',
            script: "docker tag ${env.DOCKER_IMG_TAG} ${env.DOCKER_IMG_TAG_BRANCH}")
            sh(label: 'Push Docker image name',
              script: "docker push ${env.DOCKER_IMG_TAG_BRANCH}")
        }
      }
    }
  }
  post {
    success {
      echoColor(text: '[SUCCESS]', colorfg: 'green', colorbg: 'default')
    }
    aborted {
      echoColor(text: '[ABORTED]', colorfg: 'magenta', colorbg: 'default')
    }
    failure {
      echoColor(text: '[FAILURE]', colorfg: 'red', colorbg: 'default')
    }
    unstable {
      echoColor(text: '[UNSTABLE]', colorfg: 'yellow', colorbg: 'default')
    }
  }
}

def insideGo(Closure body){
  def goAgent = docker.build("go-agent", ".ci/jenkins-go-agent")
  goAgent.inside(){
    env.HOME="${WORKSPACE}/${BASE_DIR}"
    sh(label: 'Go version', script: 'go version')
    sh(label: 'Install Mage', script: 'mage -version')
    body()
  }
}
